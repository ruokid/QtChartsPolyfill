# QtChartsPolyfill

较低版本的QtCharts功能补充或变通方案

## 1. QCategoryAxis的labels过多时会被截断显示成省略号

新版API是
[QAbstractAxis::setTruncateLabels(bool)](https://github.com/qt/qtcharts/commit/ad94166fefba448805d145d6c61a6b30209aa146)

变通方案: 自己在scene上绘制标签

```C/C++
void drawAxisLabel(QCategoryAxis *axis)
{
    // 新绘制的标签组，无须设置parent，由后面的scene接管生命周期
    auto newAxisLabels = new QGraphicsItemGroup();

    auto labels = axis->categoriesLabels(); //所有的原始标签
    for (auto item : this->scene()->items()) {
        // 开始寻找Qt绘制的标签组，这里可能需要自行修改一下条件
        if (item->childItems().size() == labels.size() + 1) {
            auto graphicsItems = item->childItems();

            for (int i = 0; i < labels.size(); i++) {
                auto textItem = static_cast<QGraphicsTextItem *>(graphicsItems[i]);
                auto newLabel = new QGraphicsTextItem(); //创建新的标签，内存由QGraphicsItemGroup管理
                newLabel->setPlainText(labels[i]); //设置成未被截断的标签
                newLabel->setPos(textItem->pos()); //设置到QCategoryAxis的标签位置
                newAxisLabels->addToGroup(newLabel);
                // 连接到QCategoryAxis的变动以便设置绘制坐标，形成跟随效果
                connect(textItem, &QGraphicsObject::xChanged, [=] { newLabel->setPos(textItem->pos()); });
            }

            break;
        }
    }

    if (!newAxisLabels->childItems().isEmpty()) {
        this->scene()->addItem(newAxisLabels); //添加绘制的标签
        axis->setLabelsVisible(false); //隐藏原始标签
    }
    else {
        delete newAxisLabels;
    }
}
```
