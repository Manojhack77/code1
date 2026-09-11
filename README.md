#include "areaselectorwidget.h"
#include <QRubberBand>
#include <QMouseEvent>
#include <QPainter>
#include <QScreen>
#include <QGuiApplication>

AreaSelectorWidget::AreaSelectorWidget(QWidget *parent) : QWidget(parent)
{
    // Make the widget frameless, topmost, translucent, and span across the screen
    setWindowFlags(Qt::FramelessWindowHint | Qt::WindowStaysOnTopHint | Qt::Tool);
    setAttribute(Qt::WA_TranslucentBackground);
    setCursor(Qt::CrossCursor);
    setGeometry(QGuiApplication::primaryScreen()->geometry());
}

void AreaSelectorWidget::mousePressEvent(QMouseEvent *event)
{
    m_origin = event->pos();
    m_rubberBand = new QRubberBand(QRubberBand::Rectangle, this);
    m_rubberBand->setGeometry(QRect(m_origin, QSize()));
    m_rubberBand->show();
}

void AreaSelectorWidget::mouseMoveEvent(QMouseEvent *event)
{
    if (m_rubberBand) {
        // .normalized() handles negative width/height if dragging backwards/upwards
        m_rubberBand->setGeometry(QRect(m_origin, event->pos()).normalized());
    }
}

void AreaSelectorWidget::mouseReleaseEvent(QMouseEvent *event)
{
    if (m_rubberBand) {
        QRect rect = m_rubberBand->geometry();
        m_rubberBand->hide();
        m_rubberBand->deleteLater();
        m_rubberBand = nullptr;

        emit areaSelected(rect); // Emit selected coordinates
    }
    close();
}

void AreaSelectorWidget::paintEvent(QPaintEvent *)
{
    QPainter painter(this);
    // Draw a semi-transparent dark tint over the screen background
    painter.fillRect(rect(), QColor(0, 0, 0, 80));
}
#ifndef AREASELECTORWIDGET_H
#define AREASELECTORWIDGET_H

#include <QWidget>
#include <QRect>

class QRubberBand;

class AreaSelectorWidget : public QWidget
{
    Q_OBJECT
public:
    explicit AreaSelectorWidget(QWidget *parent = nullptr);

signals:
    void areaSelected(const QRect &rect);

protected:
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;
    void mouseReleaseEvent(QMouseEvent *event) override;
    void paintEvent(QPaintEvent *event) override;

private:
    QPoint m_origin;
    QRubberBand *m_rubberBand = nullptr;
};

#endif // AREASELECTORWIDGET_H
