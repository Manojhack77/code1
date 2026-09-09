/**
 * @file recordreplay.cpp
 * @brief Implementation of RecordReplay application
 * @author RecordReplay Team
 * @version 1.0.0
 * @date 2024
 *
 * This file implements screen recording functionality using Qt 5.1+ framework and FFmpeg.
 */

#include "recordreplay.h"
#include "ui_recordreplay.h"
#include <QFileDialog>
#include <QMessageBox>
#include <QDir>
#include <QDebug>
#include <QStandardPaths>
#include <QTextStream>
#include <QFileInfo>

// ============================================================
// CONSTRUCTOR
// ============================================================

/**
 * @brief RecordReplay Constructor
 *
 * Initializes:
 * - UI components from .ui file
 * - FFmpeg process for screen capture
 * - Signal/slot connections
 * - Default UI states
 */
RecordReplay::RecordReplay(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::RecordReplay)
    , screenCaptureProcess(new QProcess(this))
{
    // ========== Setup UI ==========
    ui->setupUi(this);

    // ========== Configure FFmpeg Process ==========
    // Connect error and finished signals for proper status handling
    connect(screenCaptureProcess, QOverload<QProcess::ProcessError>::of(&QProcess::error),
            this, &RecordReplay::onFFmpegError);
    connect(screenCaptureProcess, QOverload<int, QProcess::ExitStatus>::of(&QProcess::finished),
            this, &RecordReplay::onFFmpegFinished);

    // Enable reading FFmpeg debug output
    connect(screenCaptureProcess, &QProcess::readyReadStandardError, [=]() {
        QString errorOutput = QString::fromUtf8(screenCaptureProcess->readAllStandardError());
        qDebug() << "FFmpeg Log:" << errorOutput;
    });

    // ========== Initialize Button States ==========
    ui->pushButton->setEnabled(true);         // Start Recording
    ui->pushButton_Record_Stop->setEnabled(false); // Stop Recording (disabled initially)

    // ========== Window Setup ==========
    this->setWindowTitle("Screen Recorder");
    this->setMinimumSize(200, 200);

    qDebug() << "RecordReplay application initialized successfully";
}

// ============================================================
// DESTRUCTOR
// ============================================================

/**
 * @brief RecordReplay Destructor
 *
 * Ensures all resources are properly released:
 * - Stops FFmpeg process if running
 * - Deletes UI resources
 */
RecordReplay::~RecordReplay()
{
    // ========== Stop Recording if Active ==========
    if (screenCaptureProcess->state() == QProcess::Running) {
        // Attempt graceful shutdown first
        screenCaptureProcess->write("q");

        // Wait up to 2 seconds for graceful termination
        if (!screenCaptureProcess->waitForFinished(5000)) {
            // If still running, forcefully kill to prevent zombie process
            screenCaptureProcess->kill();
            screenCaptureProcess->waitForFinished();
        }
    }

    // ========== Clean Up UI ==========
    delete ui;

    qDebug() << "RecordReplay application destroyed and cleaned up";
}

// ============================================================
// SECTION 1: SCREEN RECORDING IMPLEMENTATION
// ============================================================

/**
 * @brief Browse for output directory (Recording)
 *
 * Opens a directory selection dialog and stores the path
 * in lineEdit_2 for use as recording output directory.
 */
void RecordReplay::on_Browse_clicked()
{
    QString dir = QFileDialog::getExistingDirectory(
        this,
        tr("Select Output Directory for Recording"),
        QStandardPaths::writableLocation(QStandardPaths::DocumentsLocation),
        QFileDialog::ShowDirsOnly | QFileDialog::DontResolveSymlinks
    );

    if (!dir.isEmpty()) {
        ui->lineEdit_2->setText(dir);
        qDebug() << "Recording output directory selected:" << dir;
    }
}

/**
 * @brief Start screen recording
 *
 * Validates inputs and starts FFmpeg process for desktop capture.
 * Updated to initialize fragment tracking for pause/resume functionality.
 *
 * Button: pushButton (Start Recording)
 */
void RecordReplay::on_pushButton_clicked()
{
    // ========== Validate Inputs ==========
    QString fileName = ui->lineEdit->text().trimmed();
    QString filePath = ui->lineEdit_2->text().trimmed();

    if (fileName.isEmpty()) {
        QMessageBox::warning(this, "Validation Error",
            "Please enter a file name for the recording.");
        return;
    }

    if (filePath.isEmpty()) {
        QMessageBox::warning(this, "Validation Error",
            "Please select an output directory for the recording.");
        return;
    }

    // ========== Build Full Output Path & Fragment Variables ==========
    m_baseOutputPath = filePath + QDir::separator() + fileName + ".mp4";

    // Check if file already exists
    if (QFile::exists(m_baseOutputPath)) {
        QMessageBox::warning(this, "File Exists",
            "This file already exists. Please choose a different name.");
        return;
    }

    // Reset fragment tracking variables for new recording session
    m_segmentIndex = 1;
    m_isPaused = false;
    m_recordedSegments.clear();

    // Construct path for Segment 1 (e.g., C:/Videos/myvideo_part1.mp4)
    QString firstSegmentPath = QString("%1/%2_part1.mp4").arg(filePath, fileName);
    m_recordedSegments.append(firstSegmentPath);

    // ========== Get Platform-Specific FFmpeg Arguments ==========
    QStringList arguments = getFFmpegArguments(firstSegmentPath);

    // ========== Locate Local FFmpeg Binary ==========
    QString ffmpegPath = QCoreApplication::applicationDirPath() + QDir::separator();
#ifdef Q_OS_WIN
    ffmpegPath += "ffmpeg.exe";
#else
    ffmpegPath += "ffmpeg";
#endif

    // Fallback to system PATH if local file doesn't exist
    if (!QFile::exists(ffmpegPath)) {
        ffmpegPath = "ffmpeg";
    }

    // ========== Start FFmpeg Process ==========
    qDebug() << "Starting FFmpeg segment 1 from:" << ffmpegPath;
    screenCaptureProcess->start(ffmpegPath, arguments);

    // Verify FFmpeg process started successfully
    if (!screenCaptureProcess->waitForStarted(3000)) {
        QMessageBox::critical(this, "Capture Error",
            "Failed to start screen recording.\n\n"
            "Ensure FFmpeg is placed in the application folder or installed in your system PATH.");
        return;
    }

    qDebug() << "FFmpeg segment 1 started successfully (PID:"
             << screenCaptureProcess->processId() << ")";

    // ========== Update UI Button States ==========
    ui->pushButton->setEnabled(false);         // Disable Start
    ui->pushButton_Record_Stop->setEnabled(true); // Enable Stop

    QMessageBox::information(this, "Recording Started",
        "Screen recording has started.\n\n"
        "Output: " + m_baseOutputPath);
}

/**
 * @brief Stop screen recording and merge fragments if necessary
 *
 * Stops active FFmpeg process and invokes concatenation demuxer
 * if multiple video segments were created via pause/resume cycles.
 */
void RecordReplay::on_pushButton_Record_Stop_clicked()
{
    // If currently recording and not paused, gracefully stop the active process
    if (!m_isPaused && screenCaptureProcess->state() == QProcess::Running) {
        qDebug() << "Sending graceful shutdown signal to FFmpeg...";
        screenCaptureProcess->write("q");

        // Wait up to 2 seconds for graceful termination
        if (!screenCaptureProcess->waitForFinished(2000)) {
            if (screenCaptureProcess->state() == QProcess::Running) {
                qDebug() << "Force terminating FFmpeg process...";
                screenCaptureProcess->kill();
                screenCaptureProcess->waitForFinished();
            }
        } else {
            qDebug() << "FFmpeg exited gracefully.";
        }
    }

    // ========== FRAGMENT MERGING LOGIC ==========
    // Locate FFmpeg binary for concatenation process
    QString ffmpegPath = QCoreApplication::applicationDirPath() + QDir::separator();
#ifdef Q_OS_WIN
    ffmpegPath += "ffmpeg.exe";
#else
    ffmpegPath += "ffmpeg";
#endif
    if (!QFile::exists(ffmpegPath)) ffmpegPath = "ffmpeg";

    if (m_recordedSegments.size() > 1) {
        qDebug() << "Multiple segments found. Merging video parts...";

        // Create temporary list file required by FFmpeg concat demuxer
        QString listFilePath = QDir::tempPath() + "/ffmpeg_concat_list.txt";
        QFile file(listFilePath);
        if (file.open(QIODevice::WriteOnly | QIODevice::Text)) {
            QTextStream out(&file);
            for (const QString &segment : qAsConst(m_recordedSegments)) {
                out << "file '" << segment << "'\n";
            }
            file.close();
        }

        // Run concatenation process without re-encoding (-c copy)
        QProcess concatProcess;
        QStringList concatArgs;
        concatArgs << "-f" << "concat" << "-safe" << "0"
                   << "-i" << listFilePath
                   << "-c" << "copy"
                   << m_baseOutputPath;

        concatProcess.start(ffmpegPath, concatArgs);
        concatProcess.waitForFinished(-1); // Wait until merge finishes

        // Cleanup temporary concat list file and individual segment parts
        QFile::remove(listFilePath);
        for (const QString &segment : qAsConst(m_recordedSegments)) {
            QFile::remove(segment);
        }
        qDebug() << "Segments merged successfully into:" << m_baseOutputPath;

    } else if (m_recordedSegments.size() == 1) {
        // Only 1 segment recorded (never paused), just rename/move it to final path
        if (QFile::exists(m_baseOutputPath)) {
            QFile::remove(m_baseOutputPath);
        }
        QFile::rename(m_recordedSegments.first(), m_baseOutputPath);
    }

    m_recordedSegments.clear();
    m_isPaused = false;

    // Reset UI states
    ui->pushButton->setEnabled(true);
    ui->pushButton_Record_Stop->setEnabled(false);

    QMessageBox::information(this, "Recording Stopped",
        "Screen recording has been stopped and saved to:\n" + m_baseOutputPath);
}

// ============================================================
// PRIVATE UTILITY FUNCTIONS
// ============================================================

/**
 * @brief Handle FFmpeg process errors
 * @param error The error code from QProcess
 */
void RecordReplay::onFFmpegError(QProcess::ProcessError error)
{
    QString errorMsg;

    switch (error) {
        case QProcess::FailedToStart:
            errorMsg = "Failed to start FFmpeg. Check if FFmpeg is installed and in PATH.";
            break;
        case QProcess::Crashed:
            errorMsg = "FFmpeg process crashed unexpectedly.";
            break;
        case QProcess::Timedout:
            errorMsg = "FFmpeg process operation timed out.";
            break;
        case QProcess::WriteError:
            errorMsg = "Failed to write to FFmpeg process.";
            break;
        case QProcess::ReadError:
            errorMsg = "Failed to read from FFmpeg process.";
            break;
        default:
            errorMsg = "Unknown FFmpeg error occurred.";
    }

    qDebug() << "FFmpeg Error:" << errorMsg;
    QMessageBox::critical(this, "FFmpeg Error", errorMsg);

    // Reset UI states on error
    ui->pushButton->setEnabled(true);
    ui->pushButton_Record_Stop->setEnabled(false);
}

/**
 * @brief Handle FFmpeg process completion
 * @param exitCode Exit code from FFmpeg
 * @param exitStatus Normal or crashed exit
 */
void RecordReplay::onFFmpegFinished(int exitCode, QProcess::ExitStatus exitStatus)
{
    if (exitStatus == QProcess::NormalExit) {
        qDebug() << "FFmpeg completed successfully (exit code:" << exitCode << ")";
    } else {
        qDebug() << "FFmpeg crashed (exit code:" << exitCode << ")";
    }
}

/**
 * @brief Validate input strings
 * @param text Text to validate
 * @param fieldName Name of field for error messages
 * @return true if valid, false otherwise
 */
bool RecordReplay::validateInput(const QString &text, const QString &fieldName)
{
    if (text.trimmed().isEmpty()) {
        QMessageBox::warning(this, "Validation Error",
            "Please fill in the " + fieldName + " field.");
        return false;
    }
    return true;
}

/**
 * @brief Get platform-specific FFmpeg arguments for screen capture
 *
 * @param outputPath Full path for output MP4 file
 * @return QStringList containing FFmpeg command arguments
 */
QStringList RecordReplay::getFFmpegArguments(const QString &outputPath)
{
    QStringList arguments;

#ifdef Q_OS_WIN
    arguments << "-f" << "gdigrab"
              << "-framerate" << "30"
              << "-draw_mouse" << "1"
              << "-i" << "desktop"
              << "-c:v" << "libx264"
              << "-preset" << "medium"
              << "-y"
              << outputPath;

#elif defined(Q_OS_LINUX)
    arguments << "-f" << "x11grab"
              << "-framerate" << "30"
              << "-draw_mouse" << "1"
              << "-i" << ":0"
              << "-c:v" << "libx264"
              << "-preset" << "medium"
              << "-y"
              << outputPath;

#elif defined(Q_OS_MAC)
    arguments << "-f" << "avfoundation"
              << "-framerate" << "30"
              << "-i" << "1"
              << "-c:v" << "libx264"
              << "-preset" << "medium"
              << "-y"
              << outputPath;

#else
    qWarning() << "Unknown operating system - using generic FFmpeg arguments";
    arguments << "-f" << "gdigrab"
              << "-framerate" << "30"
              << "-i" << "desktop"
              << "-y"
              << outputPath;
#endif

    return arguments;
}

/**
 * @brief Toggle Pause and Resume using the Fragment Method
 *
 * - Pause: Gracefully stops the current FFmpeg process (saves part_X.mp4).
 * - Resume: Increments segment count and spins up a new FFmpeg process for the next part.
 *
 * Button: pushButtonpauseresume
 */
void RecordReplay::on_pushButtonpauseresume_clicked()
{
    // Ensure recording is active before attempting to pause/resume
    if (ui->pushButton->isEnabled() && !ui->pushButton_Record_Stop->isEnabled()) {
        qWarning() << "Cannot pause/resume: Recording has not started.";
        return;
    }

    if (!m_isPaused) {
        // ==================== PAUSE RECORDING ====================
        if (screenCaptureProcess->state() == QProcess::Running) {
            qDebug() << "Pausing recording: Sending graceful shutdown signal to current segment...";
            screenCaptureProcess->write("q");

            if (!screenCaptureProcess->waitForFinished(3000)) {
                screenCaptureProcess->kill();
                screenCaptureProcess->waitForFinished();
            }
        }

        m_isPaused = true;
        ui->pushButtonpauseresume->setText("Resume"); // Optional: Update button label
        qDebug() << "Recording Paused. Current segment saved safely.";

    } else {
        // ==================== RESUME RECORDING ====================
        m_segmentIndex++;

        // Generate filename path for the next segment part (e.g., myvideo_part2.mp4)
        QFileInfo fileInfo(m_baseOutputPath);
        QString nextSegmentPath = QString("%1/%2_part%3.%4")
                                  .arg(fileInfo.absolutePath(),
                                       fileInfo.completeBaseName(),
                                       QString::number(m_segmentIndex),
                                       fileInfo.suffix());

        m_recordedSegments.append(nextSegmentPath);

        // Get FFmpeg arguments for the new segment
        QStringList arguments = getFFmpegArguments(nextSegmentPath);

        // Locate FFmpeg binary
        QString ffmpegPath = QCoreApplication::applicationDirPath() + QDir::separator();
#ifdef Q_OS_WIN
        ffmpegPath += "ffmpeg.exe";
#else
        ffmpegPath += "ffmpeg";
#endif
        if (!QFile::exists(ffmpegPath)) {
            ffmpegPath = "ffmpeg";
        }

        qDebug() << "Resuming recording: Starting new FFmpeg process for segment" << m_segmentIndex;
        screenCaptureProcess->start(ffmpegPath, arguments);

        if (!screenCaptureProcess->waitForStarted(3000)) {
            QMessageBox::critical(this, "Capture Error", "Failed to resume recording segment.");
            return;
        }

        m_isPaused = false;
        ui->pushButtonpauseresume->setText("Pause"); // Optional: Update button label
        qDebug() << "Recording Resumed. Writing to segment:" << nextSegmentPath;
    }
}
# code1












/**
 * @file recordreplay.h
 * @brief Header file for RecordReplay application
 * @author RecordReplay Team
 * @version 1.0.0
 * @date 2024
 *
 * This file declares the RecordReplay class for screen recording functionality using Qt and FFmpeg.
 */

#ifndef RECORDREPLAY_H
#define RECORDREPLAY_H

#include <QMainWindow>
#include <QProcess>
#include <QStringList>

QT_BEGIN_NAMESPACE
namespace Ui { class RecordReplay; }
QT_END_NAMESPACE

/**
 * @class RecordReplay
 * @brief Main window class for screen recording
 */
class RecordReplay : public QMainWindow
{
    Q_OBJECT

public:
    explicit RecordReplay(QWidget *parent = nullptr);
    ~RecordReplay();

private slots:
    void on_Browse_clicked();
    void on_pushButton_clicked();

    void onFFmpegError(QProcess::ProcessError error);
    void onFFmpegFinished(int exitCode, QProcess::ExitStatus exitStatus);



    void on_pushButton_Record_Stop_clicked();

    void on_pushButtonpauseresume_clicked();

private:
    Ui::RecordReplay *ui;
    QProcess *screenCaptureProcess;
    bool m_isPaused = false;
    int m_segmentIndex = 0;
    QString m_baseOutputPath;
    QStringList m_recordedSegments;

    bool validateInput(const QString &text, const QString &fieldName);
    QStringList getFFmpegArguments(const QString &outputPath);
};

#endif // RECORDREPLAY_H
