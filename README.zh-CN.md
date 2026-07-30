[English](README.md) | 简体中文

# hexo-plotly

这是一个与主题解耦的 Hexo Plotly 标签插件，用于按文章嵌入交互式图表。它提供：

- 按页面条件加载且每页只加载一次 Plotly 3.7.0；
- CDN 优先以及自动生成的本地回退资源；
- 按文章选择性加载 MathJax 3.2.2；
- 翻译后的多语言文章共享同一份图表实现；
- Plotly modebar 本地化；
- 可复用的控件、颜色、亮暗主题辅助函数和加载界面；
- 在 `hexo server` 和 `hexo generate --watch` 中追踪图表依赖并重新生成。

## 安装

```sh
npm install hexo-plotly
```

Hexo 会从站点依赖中自动加载名称以 `hexo-` 开头的包，不需要修改主题模板。

## 使用

在文章中启用 Plotly：

```yaml
---
plotly: true
plotly_mathjax: true # 仅当图表或外部控件使用 TeX 时启用
plotly_custom_stylesheet: /css/article-plotly.css # 可选，仅用于当前页面
---
```

`plotly_custom_stylesheet` 会经过 Hexo 的 `url_for` 处理，而且只在当前页面
加载。它位于插件默认样式和全站 `plotly.custom_stylesheet` 之后，因此页面
样式具有最终覆盖优先级。

嵌入站点仓库内可信的 JavaScript 文件：

```text
{% plotly chart-id source/graph_code/post/chart.js 420 %}
```

高度参数可以省略。图表文件可以位于 Hexo 站点根目录下的任意位置，并不强制放在
`graph_code` 中。插件会拒绝绝对路径、越出站点根目录的路径、非 JavaScript
文件，以及通过符号链接逃逸到站点之外的路径。

图表文件在生成的页面中执行，可以使用以下绑定：

- `target`：图表容器；
- `Plotly`：已加载的 Plotly API；
- `HexoPlotly`：插件提供的共享辅助函数；
- `BlogPlotly`：为了兼容旧图表而保留的 `HexoPlotly` 别名；
- `chartI18n`：当前选择的共享文案和图表专用文案。

例如：

```js
const colors = HexoPlotly.getColors();
const { container, inputs, outputs } = HexoPlotly.createRangeControls(
  target,
  [{
    key: 'amplitude',
    label: chartI18n.text.amplitude,
    mathLabel: 'A',
    min: 0,
    max: 10,
    step: 0.1,
    value: 1
  }],
  { separator: chartI18n.common.controlSeparator }
);

function render() {
  HexoPlotly.setOutput(outputs, 'amplitude', inputs.amplitude.value);
  const layout = {
    ...HexoPlotly.baseLayout(colors),
    xaxis: HexoPlotly.axis('x', {}, colors),
    yaxis: HexoPlotly.axis('y', {}, colors)
  };
  target.plotlyRenderReady = Plotly.react(
    target,
    [{ x: [0, 1], y: [0, Number(inputs.amplitude.value)] }],
    layout,
    HexoPlotly.getPlotConfig()
  );
}

inputs.amplitude.addEventListener('input', render);
render();
HexoPlotly.observeTheme(render, target);
```

图表代码会被内联到文章页面中，必须是可信代码。不要嵌入未经审查的第三方
JavaScript。

## 图表多语言

把 YAML 翻译文件放在图表代码旁边：

```text
source/graph_code/post/chart.js
source/graph_code/post/chart.i18n.yml
```

```yaml
default: zh-CN
zh-CN:
  amplitude: 振幅
en:
  amplitude: Amplitude
```

每种语言必须包含完全相同的一组扁平字符串键。插件依次根据 tag 的
`lang=...` 参数、文章的 `lang`/`language` 和 Hexo 站点语言选择语言：

```text
{% plotly chart-id source/graph_code/post/chart.js 420 lang=en %}
```

JavaScript 本身不需要包含特定语言，通过 `chartI18n.text` 读取文案。插件还会
加载匹配的 Plotly 官方语言包以翻译 modebar；英语不需要额外的语言脚本。

## 配置

以下字段均为可选：

```yaml
plotly:
  asset_dir: assets/hexo-plotly
  cdn_url: https://cdn.jsdelivr.net/npm/plotly.js@3.7.0/dist/plotly.min.js
  local_url:
  mathjax_cdn_url: https://cdn.jsdelivr.net/npm/mathjax@3.2.2/es5/tex-svg.js
  mathjax_local_url:
  locale_cdn_template: https://cdn.plot.ly/plotly-locale-{locale}-latest.js
  runtime_url:
  stylesheet_url:
  custom_stylesheet: /css/plotly-overrides.css
  timeout_ms: 10000
  watch: true
  require_front_matter: true
  stop_input_propagation: false
  filter_priority: 10
  i18n:
    en:
      loading: Loading chart...
```

`local_url` 和 `mathjax_local_url` 默认指向由插件生成、与依赖版本一致的本地
资源。把某个 URL 字段设置为 `false` 可以禁用对应来源。Plotly 按以下顺序
加载：

1. 在 `timeout_ms` 内请求配置的 CDN；
2. CDN 失败或超时后，在相同超时时间内请求本地回退文件；
3. 两者都失败后再次请求 CDN，最后一次不设置超时限制。

MathJax 和语言包采用 CDN 优先、本地回退。`asset_dir` 是 Hexo 的生成路由，
并不要求源文件位于主题目录中；它也会尊重 `/en/` 这样的非根目录 `root`
配置。

`stylesheet_url` 用于替换插件默认样式表的 URL。`custom_stylesheet` 则会在
默认样式表之后额外加载一份 CSS，是覆盖颜色和适配主题时推荐使用的配置。
本地路径会经过 Hexo 的 `url_for` 处理，因此站点部署在 `/notes/` 等子路径时，
`/css/plotly-overrides.css` 仍能得到正确地址；也可以填写完整的外部 URL。
省略该字段或将其设置为 `false` 即可禁用附加样式表。

`stop_input_propagation` 用于兼容会监听滑块冒泡 `input` 事件的主题，大多数
主题应保持为 `false`。

## 主题与样式

默认样式的插件源码位于
[`assets/hexo-plotly.css`](assets/hexo-plotly.css)。通过 npm 安装后，它位于：

```text
node_modules/hexo-plotly/assets/hexo-plotly.css
```

Hexo 会把它生成为：

```text
<root>/<asset_dir>/hexo-plotly.css
```

在默认配置下，对应网站路径为：

```text
/assets/hexo-plotly/hexo-plotly.css
```

插件不会导入或修改 Hexo 主题。默认 CSS 会识别常见的 `data-theme`、
`data-color-mode`、`data-scheme`、`.dark` 和 `.light` 标志，并支持
`prefers-color-scheme`。主题不需要复制整份默认 CSS，只需覆盖
`--hexo-plotly-*` 变量。

例如，在 Hexo 站点中新建：

```text
source/css/plotly-overrides.css
```

然后配置：

```yaml
plotly:
  custom_stylesheet: /css/plotly-overrides.css
```

覆盖文件可以写成：

```css
:root {
  --hexo-plotly-text: #363636;
  --hexo-plotly-grid: rgba(0, 0, 0, 0.12);
  --hexo-plotly-primary: #1f77b4;
  --hexo-plotly-warning: #d48806;
  --hexo-plotly-accent: #6f42c1;
  --hexo-plotly-control-bg: rgba(255, 255, 255, 0.45);
  --hexo-plotly-control-border: rgba(0, 0, 0, 0.16);
}

:root[data-theme='dark'] {
  --hexo-plotly-text: #d8d8d8;
  --hexo-plotly-grid: rgba(255, 255, 255, 0.14);
  --hexo-plotly-primary: #49b1f5;
  --hexo-plotly-warning: #f6c344;
  --hexo-plotly-accent: #9b8afb;
  --hexo-plotly-control-bg: rgba(255, 255, 255, 0.04);
  --hexo-plotly-control-border: rgba(255, 255, 255, 0.18);
}
```

插件会确保自定义样式表位于默认样式表之后。通过
`HexoPlotly.getColors()` 获取颜色的图表会使用覆盖后的变量。如果图表需要在
网页实时切换亮暗主题后重新绘制，应调用 `HexoPlotly.observeTheme()`。

如果只有某篇文章需要不同样式，可以再创建
`source/css/article-plotly.css`，然后在该文章 Front Matter 中指定：

```yaml
plotly: true
plotly_custom_stylesheet: /css/article-plotly.css
```

最终加载顺序依次是插件默认 CSS、全站自定义 CSS、页面专用 CSS。如果页面
配置与全站配置指向同一 URL，插件只会加载一次。

插件不依赖主题专用的 Hexo 辅助函数、Font Awesome 图标或主题自己的国际化
注册表。

## 实时预览

在 `hexo server` 或 `hexo generate --watch` 中，插件会追踪 Post/Page 引用的
每个图表 JavaScript 文件及其相邻 `.i18n.yml` 文件。依赖发生变化后，对应
文章的渲染缓存会失效，使 Hexo 不会复用过时的内联 HTML。

位于 `source_dir` 内的文件由 Hexo 自带的 source watcher 处理。站点根目录内
其他位置的引用文件由插件按需、窄范围监听。由于 Hexo 公开的 Box API 只直接
覆盖 source/theme 目录，外部目录通知使用了 Hexo 当前的内部重新生成事件；
测试会覆盖这部分逻辑，但它也是最容易受 Hexo 大版本变化影响的部分。

## 开发

```sh
npm test
npm run check
npm pack --dry-run
```

插件要求 Node.js 20.19 或更高版本，并支持 Hexo 7 和 Hexo 8。
