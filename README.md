# QtChartsPolyfill

较低版本的QtCharts功能补充或变通方案

## 1. QCategoryAxis的labels过多时会被截断显示成省略号

新版API是
[QAbstractAxis::setTruncateLabels(bool)](https://github.com/qt/qtcharts/commit/ad94166fefba448805d145d6c61a6b30209aa146)

变通方案: 自己在scene上绘制标签

```C
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

## 2. 控件国际化问题（文本翻译）
> 注意：在Qt框架中基本上所有文本都在`tr()`函数内，
> 所以最简单处理方式为使用Qt框架提供的ts文件，将其载入Translator即可，
> 以下为不使用翻译文件时的变通方案
### 2.1 QFileDialog在QFileDialog::DontUseNativeDialog状态下需要处理的文本
本质问题是QFileSystemModel的表头文本需要处理
```C
class HeaderProxyModel : public QIdentityProxyModel
{
public:
    HeaderProxyModel(QObject *parent = nullptr) : QIdentityProxyModel(parent) {}

    QVariant headerData(int section, Qt::Orientation orientation, int role = Qt::DisplayRole) const override
    {
        if (orientation == Qt::Orientation::Horizontal && role == Qt::DisplayRole) {
            switch (section) {
                case 0: return tr("名称");
                case 1: return tr("大小");
                case 2: return tr("类型");
                case 3: return tr("修改时间");
            }
        }
        return QIdentityProxyModel::headerData(section, orientation, role);
    }
};

/**
 * @brief 显示事件
 * 必须继承自QFileDialog然后重写showEvent才能实现表头和右键菜单的
 * @param event 
 */
void FileDialog::showEvent(QShowEvent *event)
{
    auto headerModel = new HeaderProxyModel(this->pv);
    this->pv->setProxyModel(headerModel);
    auto action1 = this->pv->findChild<QAction *>("qt_rename_action");
    if (action1 != nullptr) { action1->setText(tr("重命名")); }
    auto action2 = this->pv->findChild<QAction *>("qt_delete_action");
    if (action2 != nullptr) { action2->setText(tr("删除")); }
    auto action3 = this->pv->findChild<QAction *>("qt_show_hidden_action");
    if (action3 != nullptr) { action3->setText(tr("显示隐藏文件")); }
    auto action4 = this->pv->findChild<QAction *>("qt_new_folder_action");
    if (action4 != nullptr) { action4->setText(tr("新建文件夹")); }

    // 以下5行放在构造函数里也可以
    this->setLabelText(QFileDialog::DialogLabel::Accept, tr("打开"));
    this->setLabelText(QFileDialog::DialogLabel::FileName, tr("文件名"));
    this->setLabelText(QFileDialog::DialogLabel::FileType, tr("文件类型"));
    this->setLabelText(QFileDialog::DialogLabel::LookIn, tr("位置"));
    this->setLabelText(QFileDialog::DialogLabel::Reject, tr("取消"));

    QDialog::showEvent(event);
}
```
