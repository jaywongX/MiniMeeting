# API 扩展指南

> MiniMeeting 扩展开发指南，包括插件架构、自定义编解码器、界面定制等。

---

## 1. 插件架构设计

### 1.1 插件系统概述

```
┌──────────────────────────────────────────────────────────────────────┐
│                       插件架构设计                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                        主应用程序                              │  │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │  │
│  │  │   核心    │  │  插件    │  │  事件    │  │  配置    │  │  │
│  │  │   引擎    │  │  管理器  │  │  总线    │  │  管理    │  │  │
│  │  └───────────┘  └─────┬─────┘  └───────────┘  └───────────┘  │  │
│  └─────────────────────────┼─────────────────────────────────────┘  │
│                            │                                         │
│           ┌────────────────┼────────────────┐                       │
│           │                │                │                        │
│           ▼                ▼                ▼                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │ 视频滤镜插件 │  │ 音频处理插件 │  │ UI 扩展插件  │                  │
│  │             │  │             │  │             │                  │
│  │ - 美颜滤镜  │  │ - 变声效果  │  │ - 主题定制  │                  │
│  │ - 虚拟背景  │  │ - 音效增强  │  │ - 布局扩展  │                  │
│  └─────────────┘  └─────────────┘  └─────────────┘                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 插件接口定义

```cpp
// plugin_interface.h

#include <QtPlugin>
#include <QObject>
#include <QString>
#include <QJsonObject>
#include <QImage>

/**
 * @brief 插件元信息
 */
struct PluginMetadata {
    QString id;              // 插件唯一标识
    QString name;            // 插件名称
    QString version;         // 版本号
    QString author;          // 作者
    QString description;     // 描述
    QString minAppVersion;   // 最低应用版本
    QStringList dependencies; // 依赖的其他插件
};

/**
 * @brief 插件基类接口
 */
class IPlugin : public QObject {
    Q_OBJECT
    
public:
    virtual ~IPlugin() = default;
    
    /**
     * @brief 获取插件元信息
     */
    virtual PluginMetadata metadata() const = 0;
    
    /**
     * @brief 初始化插件
     * @param config 配置参数
     * @return 是否初始化成功
     */
    virtual bool initialize(const QJsonObject& config) = 0;
    
    /**
     * @brief 关闭插件
     */
    virtual void shutdown() = 0;
    
    /**
     * @brief 获取插件配置界面
     * @return 配置界面的 Widget，如果没有则返回 nullptr
     */
    virtual QWidget* createSettingsWidget() { return nullptr; }
    
signals:
    /**
     * @brief 插件错误信号
     */
    void errorOccurred(const QString& error);
    
    /**
     * @brief 插件状态变化信号
     */
    void statusChanged(const QString& status);
};

/**
 * @brief 视频处理插件接口
 */
class IVideoFilterPlugin : public IPlugin {
    Q_OBJECT
    
public:
    /**
     * @brief 处理视频帧
     * @param frame 输入视频帧
     * @return 处理后的视频帧
     */
    virtual QImage processFrame(const QImage& frame) = 0;
    
    /**
     * @brief 是否启用
     */
    virtual bool isEnabled() const = 0;
    
    /**
     * @brief 设置启用状态
     */
    virtual void setEnabled(bool enabled) = 0;
    
    /**
     * @brief 获取支持的像素格式
     */
    virtual QList<QImage::Format> supportedFormats() const {
        return { QImage::Format_RGB32, QImage::Format_ARGB32 };
    }
};

/**
 * @brief 音频处理插件接口
 */
class IAudioFilterPlugin : public IPlugin {
    Q_OBJECT
    
public:
    /**
     * @brief 处理音频数据
     * @param data 输入音频数据
     * @param sampleRate 采样率
     * @param channels 通道数
     * @return 处理后的音频数据
     */
    virtual QByteArray processAudio(const QByteArray& data, 
                                    int sampleRate, 
                                    int channels) = 0;
    
    /**
     * @brief 获取延迟（采样数）
     */
    virtual int getLatency() const { return 0; }
};

/**
 * @brief UI 扩展插件接口
 */
class IUIExtensionPlugin : public IPlugin {
    Q_OBJECT
    
public:
    /**
     * @brief 获取扩展的工具栏按钮
     */
    virtual QList<QAction*> getToolbarActions() { return {}; }
    
    /**
     * @brief 获取扩展的菜单项
     */
    virtual QList<QAction*> getMenuActions() { return {}; }
    
    /**
     * @brief 获取扩展的面板
     */
    virtual QWidget* getPanel(const QString& location) { return nullptr; }
    
    /**
     * @brief 获取扩展的设置页面
     */
    virtual QList<SettingsPage> getSettingsPages() { return {}; }
};

Q_DECLARE_INTERFACE(IVideoFilterPlugin, "com.minimeeting.IVideoFilterPlugin/1.0")
Q_DECLARE_INTERFACE(IAudioFilterPlugin, "com.minimeeting.IAudioFilterPlugin/1.0")
Q_DECLARE_INTERFACE(IUIExtensionPlugin, "com.minimeeting.IUIExtensionPlugin/1.0")
```

### 1.3 插件管理器

```cpp
// plugin_manager.h

#include <QObject>
#include <QMap>
#include <QDir>
#include <QPluginLoader>

/**
 * @brief 插件管理器
 */
class PluginManager : public QObject {
    Q_OBJECT
    
public:
    static PluginManager* instance();
    
    /**
     * @brief 加载所有插件
     * @param pluginDir 插件目录
     */
    void loadPlugins(const QString& pluginDir);
    
    /**
     * @brief 卸载所有插件
     */
    void unloadPlugins();
    
    /**
     * @brief 获取插件
     */
    template<typename T>
    T* getPlugin(const QString& id) {
        if (m_plugins.contains(id)) {
            return qobject_cast<T*>(m_plugins[id]->instance());
        }
        return nullptr;
    }
    
    /**
     * @brief 获取所有插件
     */
    template<typename T>
    QList<T*> getPlugins() {
        QList<T*> result;
        for (auto* plugin : m_plugins) {
            T* casted = qobject_cast<T*>(plugin->instance());
            if (casted) {
                result.append(casted);
            }
        }
        return result;
    }
    
    /**
     * @brief 获取插件列表
     */
    QList<PluginMetadata> getPluginList() const;
    
signals:
    void pluginLoaded(const QString& id);
    void pluginUnloaded(const QString& id);
    void pluginError(const QString& id, const QString& error);
    
private:
    PluginManager() = default;
    
    QMap<QString, QPluginLoader*> m_plugins;
};

// plugin_manager.cpp

PluginManager* PluginManager::instance() {
    static PluginManager manager;
    return &manager;
}

void PluginManager::loadPlugins(const QString& pluginDir) {
    QDir dir(pluginDir);
    
    for (const QString& fileName : dir.entryList(QDir::Files)) {
        QString filePath = dir.absoluteFilePath(fileName);
        
        QPluginLoader* loader = new QPluginLoader(filePath);
        
        // 检查元数据
        QJsonObject metadata = loader->metaData()["MetaData"].toObject();
        QString id = metadata["id"].toString();
        
        if (id.isEmpty() || m_plugins.contains(id)) {
            qWarning() << "Invalid or duplicate plugin:" << filePath;
            delete loader;
            continue;
        }
        
        // 加载插件
        if (!loader->load()) {
            qWarning() << "Failed to load plugin:" << loader->errorString();
            delete loader;
            continue;
        }
        
        // 初始化插件
        IPlugin* plugin = qobject_cast<IPlugin*>(loader->instance());
        if (!plugin) {
            qWarning() << "Invalid plugin interface:" << filePath;
            loader->unload();
            delete loader;
            continue;
        }
        
        // 检查版本兼容性
        QString minVersion = plugin->metadata().minAppVersion;
        if (!isVersionCompatible(minVersion, APP_VERSION)) {
            qWarning() << "Plugin version incompatible:" << id;
            loader->unload();
            delete loader;
            continue;
        }
        
        // 初始化
        QJsonObject config = loadPluginConfig(id);
        if (!plugin->initialize(config)) {
            qWarning() << "Failed to initialize plugin:" << id;
            loader->unload();
            delete loader;
            continue;
        }
        
        m_plugins[id] = loader;
        emit pluginLoaded(id);
        qInfo() << "Plugin loaded:" << id;
    }
}

void PluginManager::unloadPlugins() {
    for (auto it = m_plugins.begin(); it != m_plugins.end(); ++it) {
        IPlugin* plugin = qobject_cast<IPlugin*>(it.value()->instance());
        if (plugin) {
            plugin->shutdown();
        }
        it.value()->unload();
        delete it.value();
        emit pluginUnloaded(it.key());
    }
    m_plugins.clear();
}
```

---

## 2. 视频处理扩展

### 2.1 视频滤镜插件示例

```cpp
// beauty_filter_plugin.h

#include "plugin_interface.h"
#include <QSlider>
#include <QSpinBox>

/**
 * @brief 美颜滤镜插件
 */
class BeautyFilterPlugin : public QObject, public IVideoFilterPlugin {
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.minimeeting.IVideoFilterPlugin/1.0")
    Q_INTERFACES(IVideoFilterPlugin)
    
public:
    PluginMetadata metadata() const override {
        return {
            "beauty-filter",
            "美颜滤镜",
            "1.0.0",
            "MiniMeeting Team",
            "实时美颜效果，支持磨皮、美白、瘦脸",
            "1.0.0",
            {}
        };
    }
    
    bool initialize(const QJsonObject& config) override {
        m_smoothLevel = config.value("smoothLevel", 5).toInt();
        m_whitenLevel = config.value("whitenLevel", 5).toInt();
        m_enabled = config.value("enabled", true).toBool();
        return true;
    }
    
    void shutdown() override {
        // 清理资源
    }
    
    QImage processFrame(const QImage& frame) override {
        if (!m_enabled) {
            return frame;
        }
        
        QImage result = frame.convertToFormat(QImage::Format_RGB32);
        
        // 应用磨皮效果
        applySmooth(result, m_smoothLevel);
        
        // 应用美白效果
        applyWhiten(result, m_whitenLevel);
        
        return result;
    }
    
    bool isEnabled() const override { return m_enabled; }
    void setEnabled(bool enabled) override { m_enabled = enabled; }
    
    QWidget* createSettingsWidget() override {
        QWidget* widget = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(widget);
        
        // 磨皮级别
        QSlider* smoothSlider = new QSlider(Qt::Horizontal);
        smoothSlider->setRange(0, 10);
        smoothSlider->setValue(m_smoothLevel);
        connect(smoothSlider, &QSlider::valueChanged, [this](int value) {
            m_smoothLevel = value;
        });
        layout->addWidget(new QLabel("磨皮级别:"));
        layout->addWidget(smoothSlider);
        
        // 美白级别
        QSlider* whitenSlider = new QSlider(Qt::Horizontal);
        whitenSlider->setRange(0, 10);
        whitenSlider->setValue(m_whitenLevel);
        connect(whitenSlider, &QSlider::valueChanged, [this](int value) {
            m_whitenLevel = value;
        });
        layout->addWidget(new QLabel("美白级别:"));
        layout->addWidget(whitenSlider);
        
        return widget;
    }
    
private:
    void applySmooth(QImage& image, int level) {
        // 双边滤波实现磨皮
        int radius = level * 2 + 1;
        // ... 实现细节
    }
    
    void applyWhiten(QImage& image, int level) {
        // 调整亮度和对比度
        // ... 实现细节
    }
    
    bool m_enabled = true;
    int m_smoothLevel = 5;
    int m_whitenLevel = 5;
};
```

### 2.2 虚拟背景插件

```cpp
// virtual_background_plugin.h

class VirtualBackgroundPlugin : public QObject, public IVideoFilterPlugin {
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.minimeeting.IVideoFilterPlugin/1.0")
    Q_INTERFACES(IVideoFilterPlugin)
    
public:
    PluginMetadata metadata() const override {
        return {
            "virtual-background",
            "虚拟背景",
            "1.0.0",
            "MiniMeeting Team",
            "用图片或视频替换会议背景",
            "1.0.0",
            {}
        };
    }
    
    bool initialize(const QJsonObject& config) override {
        QString bgPath = config.value("backgroundPath").toString();
        if (!bgPath.isEmpty()) {
            loadBackground(bgPath);
        }
        return true;
    }
    
    QImage processFrame(const QImage& frame) override {
        if (!m_enabled || m_background.isNull()) {
            return frame;
        }
        
        // 使用 AI 分割前景和背景
        cv::Mat mask = segmentPerson(frame);
        
        // 合成结果
        return compositeFrame(frame, m_background, mask);
    }
    
    QWidget* createSettingsWidget() override {
        QWidget* widget = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(widget);
        
        // 背景选择
        QPushButton* selectBtn = new QPushButton("选择背景图片");
        connect(selectBtn, &QPushButton::clicked, [this]() {
            QString file = QFileDialog::getOpenFileName(nullptr,
                "选择背景图片", "", "Images (*.png *.jpg)");
            if (!file.isEmpty()) {
                loadBackground(file);
            }
        });
        layout->addWidget(selectBtn);
        
        // 预览
        QLabel* previewLabel = new QLabel();
        previewLabel->setFixedSize(320, 180);
        if (!m_background.isNull()) {
            previewLabel->setPixmap(QPixmap::fromImage(
                m_background.scaled(320, 180, Qt::KeepAspectRatio)));
        }
        layout->addWidget(previewLabel);
        
        // 预设背景
        QComboBox* presetCombo = new QComboBox();
        presetCombo->addItem("自定义");
        presetCombo->addItem("办公室");
        presetCombo->addItem("海滩");
        presetCombo->addItem("山脉");
        presetCombo->addItem("模糊");
        connect(presetCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
            [this, previewLabel](int index) {
                if (index > 0) {
                    loadPresetBackground(index);
                }
            });
        layout->addWidget(new QLabel("预设背景:"));
        layout->addWidget(presetCombo);
        
        return widget;
    }
    
private:
    void loadBackground(const QString& path) {
        m_background = QImage(path);
        if (m_background.isNull()) {
            emit errorOccurred("Failed to load background image");
        }
    }
    
    void loadPresetBackground(int index) {
        QString presetPath = QString(":/backgrounds/bg_%1.jpg").arg(index);
        m_background = QImage(presetPath);
    }
    
    cv::Mat segmentPerson(const QImage& frame) {
        // 使用深度学习模型进行人像分割
        // 可以集成 TensorFlow/PyTorch 模型
        // ...
    }
    
    QImage compositeFrame(const QImage& frame, const QImage& bg, const cv::Mat& mask) {
        // 根据 mask 合成前景和背景
        // ...
    }
    
    QImage m_background;
    bool m_enabled = true;
};
```

---

## 3. 音频处理扩展

### 3.1 音频滤镜插件示例

```cpp
// voice_changer_plugin.h

class VoiceChangerPlugin : public QObject, public IAudioFilterPlugin {
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.minimeeting.IAudioFilterPlugin/1.0")
    Q_INTERFACES(IAudioFilterPlugin)
    
public:
    enum class VoiceType {
        Normal,
        Deep,
        High,
        Robot,
        Echo
    };
    
    PluginMetadata metadata() const override {
        return {
            "voice-changer",
            "变声器",
            "1.0.0",
            "MiniMeeting Team",
            "实时变声效果，支持多种声音风格",
            "1.0.0",
            {}
        };
    }
    
    bool initialize(const QJsonObject& config) override {
        m_voiceType = static_cast<VoiceType>(
            config.value("voiceType", 0).toInt());
        m_pitchShift = config.value("pitchShift", 0.0).toDouble();
        return true;
    }
    
    QByteArray processAudio(const QByteArray& data, 
                           int sampleRate, 
                           int channels) override
    {
        if (m_voiceType == VoiceType::Normal) {
            return data;
        }
        
        // 转换为浮点采样
        std::vector<float> samples = toFloatSamples(data);
        
        switch (m_voiceType) {
            case VoiceType::Deep:
                applyPitchShift(samples, sampleRate, -3.0);
                break;
            case VoiceType::High:
                applyPitchShift(samples, sampleRate, 3.0);
                break;
            case VoiceType::Robot:
                applyRobotVoice(samples, sampleRate);
                break;
            case VoiceType::Echo:
                applyEcho(samples, sampleRate, 0.3, 0.5);
                break;
        }
        
        return toByteArray(samples);
    }
    
    QWidget* createSettingsWidget() override {
        QWidget* widget = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(widget);
        
        // 变声类型选择
        QComboBox* typeCombo = new QComboBox();
        typeCombo->addItem("正常", static_cast<int>(VoiceType::Normal));
        typeCombo->addItem("低沉", static_cast<int>(VoiceType::Deep));
        typeCombo->addItem("尖细", static_cast<int>(VoiceType::High));
        typeCombo->addItem("机器人", static_cast<int>(VoiceType::Robot));
        typeCombo->addItem("回声", static_cast<int>(VoiceType::Echo));
        connect(typeCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
            [this](int index) {
                m_voiceType = static_cast<VoiceType>(index);
            });
        layout->addWidget(new QLabel("变声类型:"));
        layout->addWidget(typeCombo);
        
        // 音调微调
        QSlider* pitchSlider = new QSlider(Qt::Horizontal);
        pitchSlider->setRange(-12, 12);
        pitchSlider->setValue(0);
        connect(pitchSlider, &QSlider::valueChanged, [this](int value) {
            m_pitchShift = value / 2.0;
        });
        layout->addWidget(new QLabel("音调微调:"));
        layout->addWidget(pitchSlider);
        
        return widget;
    }
    
private:
    void applyPitchShift(std::vector<float>& samples, 
                        int sampleRate, 
                        double semitones)
    {
        // 使用相位声码器算法实现音调变化
        // ...
    }
    
    void applyRobotVoice(std::vector<float>& samples, int sampleRate) {
        // 添加机械感：环形调制 + 量化
        double carrierFreq = 50.0;
        for (size_t i = 0; i < samples.size(); i++) {
            double t = i / static_cast<double>(sampleRate);
            samples[i] *= std::sin(2 * M_PI * carrierFreq * t);
            // 量化到低比特
            samples[i] = std::round(samples[i] * 16) / 16;
        }
    }
    
    void applyEcho(std::vector<float>& samples, 
                   int sampleRate, 
                   double delay, 
                   double decay)
    {
        int delaySamples = static_cast<int>(delay * sampleRate);
        std::vector<float> echoed(samples.size());
        
        for (size_t i = 0; i < samples.size(); i++) {
            echoed[i] = samples[i];
            if (i >= static_cast<size_t>(delaySamples)) {
                echoed[i] += decay * samples[i - delaySamples];
            }
        }
        
        samples = std::move(echoed);
    }
    
    VoiceType m_voiceType = VoiceType::Normal;
    double m_pitchShift = 0.0;
};
```

---

## 4. 自定义编解码器扩展

### 4.1 编解码器接口

```cpp
// codec_interface.h

/**
 * @brief 视频编解码器接口
 */
class IVideoCodec {
public:
    virtual ~IVideoCodec() = default;
    
    /**
     * @brief 获取编解码器名称
     */
    virtual std::string name() const = 0;
    
    /**
     * @brief 获取编解码器描述
     */
    virtual std::string description() const = 0;
    
    /**
     * @brief 编码视频帧
     */
    virtual bool encode(const VideoFrame& frame, 
                       std::vector<uint8_t>& output) = 0;
    
    /**
     * @brief 解码视频帧
     */
    virtual bool decode(const std::vector<uint8_t>& input,
                       VideoFrame& frame) = 0;
    
    /**
     * @brief 设置编码参数
     */
    virtual void setBitrate(int bitrate) = 0;
    virtual void setFrameRate(int fps) = 0;
    virtual void setResolution(int width, int height) = 0;
};

/**
 * @brief 编解码器工厂
 */
class CodecFactory {
public:
    static CodecFactory* instance();
    
    /**
     * @brief 注册编解码器
     */
    void registerCodec(const std::string& name,
                       std::function<std::unique_ptr<IVideoCodec>()> creator);
    
    /**
     * @brief 创建编解码器
     */
    std::unique_ptr<IVideoCodec> createCodec(const std::string& name);
    
    /**
     * @brief 获取可用编解码器列表
     */
    std::vector<std::string> availableCodecs() const;
    
private:
    std::map<std::string, std::function<std::unique_ptr<IVideoCodec>()>> m_creators;
};
```

### 4.2 自定义编解码器示例

```cpp
// custom_video_codec.h

/**
 * @brief 自定义 AV1 编解码器
 */
class AV1Codec : public IVideoCodec {
public:
    std::string name() const override { return "AV1"; }
    
    std::string description() const override {
        return "AOMedia Video 1 (AV1) codec";
    }
    
    bool encode(const VideoFrame& frame, 
                std::vector<uint8_t>& output) override
    {
        // 使用 libaom 进行 AV1 编码
        // ...
        return true;
    }
    
    bool decode(const std::vector<uint8_t>& input,
                VideoFrame& frame) override
    {
        // 使用 libaom 进行 AV1 解码
        // ...
        return true;
    }
    
    void setBitrate(int bitrate) override {
        m_bitrate = bitrate;
    }
    
    void setFrameRate(int fps) override {
        m_fps = fps;
    }
    
    void setResolution(int width, int height) override {
        m_width = width;
        m_height = height;
    }
    
private:
    int m_bitrate = 500000;
    int m_fps = 30;
    int m_width = 1280;
    int m_height = 720;
};

// 注册编解码器
REGISTER_CODEC(av1, []() {
    return std::make_unique<AV1Codec>();
});
```

---

## 5. 界面定制扩展

### 5.1 主题插件

```cpp
// theme_plugin.h

class ThemePlugin : public QObject, public IUIExtensionPlugin {
    Q_OBJECT
    Q_PLUGIN_METADATA(IID "com.minimeeting.IUIExtensionPlugin/1.0")
    Q_INTERFACES(IUIExtensionPlugin)
    
public:
    PluginMetadata metadata() const override {
        return {
            "theme-dark",
            "深色主题",
            "1.0.0",
            "MiniMeeting Team",
            "深色主题，护眼模式",
            "1.0.0",
            {}
        };
    }
    
    bool initialize(const QJsonObject& config) override {
        // 加载主题配置
        loadTheme(config.value("themeName", "dark").toString());
        return true;
    }
    
    QWidget* createSettingsWidget() override {
        QWidget* widget = new QWidget();
        QVBoxLayout* layout = new QVBoxLayout(widget);
        
        // 主题选择
        QComboBox* themeCombo = new QComboBox();
        themeCombo->addItem("深色", "dark");
        themeCombo->addItem("浅色", "light");
        themeCombo->addItem("高对比度", "high-contrast");
        
        connect(themeCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
            [this](int index) {
                applyTheme(static_cast<Theme>(index));
            });
        
        layout->addWidget(new QLabel("主题:"));
        layout->addWidget(themeCombo);
        
        return widget;
    }
    
private:
    enum class Theme { Dark, Light, HighContrast };
    
    void loadTheme(const QString& name) {
        QFile file(QString(":/themes/%1.qss").arg(name));
        if (file.open(QFile::ReadOnly)) {
            QString styleSheet = QLatin1String(file.readAll());
            qApp->setStyleSheet(styleSheet);
        }
    }
    
    void applyTheme(Theme theme) {
        switch (theme) {
            case Theme::Dark:
                qApp->setStyleSheet(loadStyleSheet(":/themes/dark.qss"));
                break;
            case Theme::Light:
                qApp->setStyleSheet(loadStyleSheet(":/themes/light.qss"));
                break;
            case Theme::HighContrast:
                qApp->setStyleSheet(loadStyleSheet(":/themes/high-contrast.qss"));
                break;
        }
    }
};
```

### 5.2 布局扩展

```cpp
// custom_layout_plugin.h

class CustomLayoutPlugin : public QObject, public IUIExtensionPlugin {
    Q_OBJECT
    
public:
    QList<QAction*> getToolbarActions() override {
        QList<QAction*> actions;
        
        // 添加自定义布局切换按钮
        QAction* gridAction = new QAction(QIcon(":/icons/grid.png"), "网格布局");
        connect(gridAction, &QAction::triggered, this, &CustomLayoutPlugin::switchToGridLayout);
        actions.append(gridAction);
        
        QAction* focusAction = new QAction(QIcon(":/icons/focus.png"), "聚焦布局");
        connect(focusAction, &QAction::triggered, this, &CustomLayoutPlugin::switchToFocusLayout);
        actions.append(focusAction);
        
        QAction* galleryAction = new QAction(QIcon(":/icons/gallery.png"), "画廊布局");
        connect(galleryAction, &QAction::triggered, this, &CustomLayoutPlugin::switchToGalleryLayout);
        actions.append(galleryAction);
        
        return actions;
    }
    
private:
    void switchToGridLayout() {
        emit layoutRequested("grid");
    }
    
    void switchToFocusLayout() {
        emit layoutRequested("focus");
    }
    
    void switchToGalleryLayout() {
        emit layoutRequested("gallery");
    }
    
signals:
    void layoutRequested(const QString& layoutName);
};
```

---

## 6. 信令扩展

### 6.1 自定义信令消息

```javascript
// 服务器端自定义消息处理
class CustomSignalingHandler {
    constructor() {
        this.messageHandlers = new Map();
    }
    
    // 注册自定义消息处理器
    registerHandler(type, handler) {
        this.messageHandlers.set(type, handler);
    }
    
    // 处理消息
    handleMessage(ws, message) {
        const { type, payload } = JSON.parse(message);
        
        const handler = this.messageHandlers.get(type);
        if (handler) {
            handler(ws, payload);
        } else {
            console.warn('Unknown message type:', type);
        }
    }
}

// 使用示例
const handler = new CustomSignalingHandler();

// 注册白板消息处理
handler.registerHandler('whiteboard_draw', (ws, payload) => {
    const { roomId, userId, action } = payload;
    const room = rooms.get(roomId);
    
    if (room) {
        // 广播给房间内其他用户
        room.broadcast({
            type: 'whiteboard_draw',
            payload: { userId, action }
        }, ws);
    }
});

// 注册投票消息处理
handler.registerHandler('poll_create', (ws, payload) => {
    const { roomId, question, options } = payload;
    const poll = createPoll(roomId, question, options);
    
    const room = rooms.get(roomId);
    if (room) {
        room.broadcast({
            type: 'poll_created',
            payload: poll
        });
    }
});

handler.registerHandler('poll_vote', (ws, payload) => {
    const { pollId, optionIndex, userId } = payload;
    
    const poll = getPoll(pollId);
    if (poll) {
        poll.vote(optionIndex, userId);
        
        // 广播投票结果
        const room = rooms.get(poll.roomId);
        room.broadcast({
            type: 'poll_updated',
            payload: poll.getResults()
        });
    }
});
```

### 6.2 数据通道扩展

```javascript
// WebRTC DataChannel 扩展
class DataChannelManager {
    constructor() {
        this.channels = new Map();
        this.handlers = new Map();
    }
    
    // 创建数据通道
    createChannel(pc, label, options = {}) {
        const channel = pc.createDataChannel(label, {
            ordered: options.ordered ?? true,
            maxRetransmits: options.maxRetransmits ?? 3
        });
        
        channel.onopen = () => {
            console.log('DataChannel opened:', label);
            this.channels.set(label, channel);
        };
        
        channel.onclose = () => {
            console.log('DataChannel closed:', label);
            this.channels.delete(label);
        };
        
        channel.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(label, data);
        };
        
        return channel;
    }
    
    // 发送数据
    send(label, data) {
        const channel = this.channels.get(label);
        if (channel && channel.readyState === 'open') {
            channel.send(JSON.stringify(data));
        }
    }
    
    // 注册消息处理器
    onMessage(label, handler) {
        if (!this.handlers.has(label)) {
            this.handlers.set(label, []);
        }
        this.handlers.get(label).push(handler);
    }
    
    handleMessage(label, data) {
        const handlers = this.handlers.get(label) || [];
        for (const handler of handlers) {
            handler(data);
        }
    }
}

// 使用示例：文件传输
const dcManager = new DataChannelManager();

// 创建文件传输通道
const fileChannel = dcManager.createChannel(pc, 'file-transfer', {
    ordered: true,
    maxRetransmits: 10
});

// 发送文件
async function sendFile(file) {
    const chunkSize = 16384;  // 16KB
    const totalChunks = Math.ceil(file.size / chunkSize);
    
    // 发送文件信息
    dcManager.send('file-transfer', {
        type: 'file_start',
        name: file.name,
        size: file.size,
        totalChunks
    });
    
    // 分块发送
    const reader = new FileReader();
    for (let i = 0; i < totalChunks; i++) {
        const start = i * chunkSize;
        const end = Math.min(start + chunkSize, file.size);
        const chunk = file.slice(start, end);
        
        const buffer = await chunk.arrayBuffer();
        dcManager.send('file-transfer', {
            type: 'file_chunk',
            index: i,
            data: Array.from(new Uint8Array(buffer))
        });
    }
    
    // 发送结束
    dcManager.send('file-transfer', {
        type: 'file_end'
    });
}

// 接收文件
dcManager.onMessage('file-transfer', async (data) => {
    switch (data.type) {
        case 'file_start':
            currentFile = {
                name: data.name,
                size: data.size,
                chunks: new Array(data.totalChunks)
            };
            break;
            
        case 'file_chunk':
            currentFile.chunks[data.index] = new Uint8Array(data.data);
            break;
            
        case 'file_end':
            // 合并所有块并保存
            const blob = new Blob(currentFile.chunks);
            saveAs(blob, currentFile.name);
            break;
    }
});
```

---

## 7. 插件开发指南

### 7.1 开发环境配置

```cmake
# CMakeLists.txt for plugin development

cmake_minimum_required(VERSION 3.16)
project(BeautyFilterPlugin)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Qt
find_package(Qt5 COMPONENTS Core Widgets REQUIRED)

# MiniMeeting SDK
set(MINIMEETING_SDK_DIR "/path/to/sdk")
include_directories(${MINIMEETING_SDK_DIR}/include)

# Plugin source
add_library(BeautyFilterPlugin SHARED
    beauty_filter_plugin.cpp
    beauty_filter_plugin.h
)

target_link_libraries(BeautyFilterPlugin
    Qt5::Core
    Qt5::Widgets
)

# Plugin metadata
set_target_properties(BeautyFilterPlugin PROPERTIES
    PREFIX ""
    OUTPUT_NAME "beauty_filter"
)

# Install
install(TARGETS BeautyFilterPlugin
    DESTINATION plugins
)
```

### 7.2 插件打包格式

```
plugin-name/
├── plugin.dll / plugin.so   # 插件库文件
├── metadata.json            # 元数据
├── resources/               # 资源文件
│   ├── icons/
│   ├── images/
│   └── models/
└── config/
    └── default.json         # 默认配置
```

```json
// metadata.json
{
    "id": "beauty-filter",
    "name": "美颜滤镜",
    "version": "1.0.0",
    "author": "MiniMeeting Team",
    "description": "实时美颜效果",
    "minAppVersion": "1.0.0",
    "dependencies": [],
    "permissions": [
        "video.process",
        "settings.ui"
    ],
    "entry": "plugin.dll",
    "config": "config/default.json"
}
```

---

## 8. 总结

### 扩展点汇总

| 扩展点 | 接口 | 用途 |
|--------|------|------|
| **视频处理** | IVideoFilterPlugin | 美颜、虚拟背景、滤镜 |
| **音频处理** | IAudioFilterPlugin | 变声、音效、降噪 |
| **UI 扩展** | IUIExtensionPlugin | 主题、布局、面板 |
| **编解码器** | IVideoCodec | 自定义编解码器 |
| **信令** | CustomSignalingHandler | 自定义消息类型 |
| **数据通道** | DataChannelManager | 文件传输、白板 |

### 开发流程

1. **定义接口**：继承基础插件接口
2. **实现功能**：编写具体业务逻辑
3. **添加配置**：提供设置界面
4. **测试验证**：确保兼容性和稳定性
5. **打包发布**：按规范打包插件
