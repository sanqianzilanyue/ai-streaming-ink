# 让回复一段段浮出来

**AI 聊天网页的流式淡墨**

2026 年 8 月 24 日，Anthropic 在 X 上发了一条：网页版和桌面版 Claude 的长回答，流式输出顺了四倍。帖子里有一段前后对比的视频，旧版是一个字一个字往外蹦，新版的字像是一段一段浮上来的。她看完就说想要。

这一篇记的是我们照着做的全过程：先让字带上淡墨，再让它按读信的速度放出来，最后把收尾接好。中间她拿手机试出了两个毛病，也一并写进来了。代码是原生 JavaScript，哪种前端框架都能套。我们自己的页面用 React，要点放在文末。

**在线效果和完整网页：** https://sanqianzilanyue.github.io/ai-streaming-ink/

---

## 三条结论

1. **新字先淡后浓，只动透明度。** 不动排版、不动高度，底下的字就不会被挤得一跳一跳。
2. **字先排队，一个字一个字慢慢放，每个字用一秒多慢慢显透。** 淡入要比两个字之间的间隔长得多，这样看不见字「冒出来」的那一下，只看见墨往前洇。
3. **写完了，队里剩下的字接着放完。** 时间和 token 那行小字，要等最后一个字显透了再出来。

---

## 01 · Anthropic 做了什么

帖子里说的是两件事。

第一件是**只重画还在长的那一截**。以前每来一个字，整条回答都要重新处理一遍；回答越长越卡，在慢一点的笔记本上会顿住好几百毫秒。新的做法把写完的段落、代码块、表格当成定了形的东西，不再碰它们，只更新还在往下长的末尾。

第二件就是她想要的那种**浮出来的样子**。我们把对比视频一帧一帧拆开量过：新来的一个词，从看不见到完全显出来，大约用了 0.2 秒，先快后慢。要说明的是，视频角上写着它是照实测数据重做的演示，Anthropic 的代码并没有公开，所以这个 0.2 秒是我们从视频里量出来的，不是从代码里看来的。

| | 新字显全要多久 | 曲线 | 一大坨字同时到的时候 |
|---|---|---|---|
| Anthropic（从视频量） | 约 0.2 秒 | 先快后慢 | 视频里没拍到 |
| 我们 | 1.68 秒 | 缓起缓收 | 排进队，一个字一个字慢慢放 |

我们故意做慢了。他们的回答里大多是代码和表格，利落一点合适；我们的聊天界面是衬线字体写的信，墨慢慢洇开才好看。

---

## 02 · 第一笔：只重画正在长的那一截

这一步是后面所有效果的地基，很多人的聊天页卡就卡在这里。

- 正在流式输出的那条回答，单独做成一个小组件。来了新字只更新它，整个消息列表不跟着重新渲染。
- 网络送来的字很碎，有时候一秒几十批。用 `requestAnimationFrame` 合帧，一帧最多更新一次：

```js
let pending = null, queued = false;
function onDelta(fullText) {          // 每收到一批字就调一次，传目前为止的全文
  pending = fullText;
  if (!queued) {
    queued = true;
    requestAnimationFrame(() => { queued = false; ink.push(pending); });
  }
}
```

---

## 03 · 第二笔：淡墨

做法很简单：显出来的字分两种，**已经落定的**是一整段普通文本，**最近一两秒内刚显出来的**几小截各套一个 `<span class="ink">`，让它从透明慢慢变不透明。淡完以后，这几截并回普通文本，`span` 删掉。这样页面上永远只挂着几个 `span`，回答再长也不会越来越重。

```css
.letter { white-space: pre-wrap; }            /* 换行直接用 \n，不用拆 <br> */
.ink { animation: ink 1.68s cubic-bezier(0.3, 0.1, 0.3, 1) both; }
@keyframes ink { from { opacity: 0; } to { opacity: 1; } }
@media (prefers-reduced-motion: reduce) { .ink { animation: none; } }
```

几个要点：

- **只动 `opacity`。** 透明度交给显卡合成，不碰排版，手机上滚动、弹键盘的时候也不掉帧。行内的 `span` 做不了位移动画；加模糊（`blur`）在手机上又太费，几十个 `span` 同时模糊会卡。
- **`both` 不能少。** 它让还没轮到的那几截一出生就是透明的，不会先闪一下实心。
- **每个 `span` 的动画延时在它出生那一刻就定死。** 如果每次重新渲染都重算一遍 `animation-delay`，浏览器会把动画的时间轴挪位，字会跳一下。

---

## 04 · 第三笔：一个字一个字，慢慢洇

只有淡墨还不够。第一版做好，她先在电脑上看了对照页，觉得好看；拿到手机上真聊了一句，就嫌字出得太快，好看劲儿一下子没了。

原因是模型写得比人读得快得多。我们这边的模型一秒能写八九十个字，有时一下送来一大坨。只加淡墨的话，字来多快就显多快，一大段一大段地往外涌，淡墨根本来不及洇开。所以要加一个队：模型写来的字先进队，再按我们定的节奏放到屏幕上。节奏怎么定，我们试了三回。

- **第一回，一个字一个字放，每秒 34 个，每个字淡入 0.56 秒。** 她觉得还是有点快，而且有一点点卡。一秒三十几下，看上去就是一台很快的打字机；聊天页又是每长出一行，屏幕就往上跳一下。
- **第二回，一小句一小句地放，每句淡入 0.75 秒。** 速度是慢下来了，可她说一句一句地跳出来，跳的那一下还是突兀：一整句同时开始淡入，「出现」的那一刻照样看得见。
- **第三回，照她最喜欢的样子做。** 她最喜欢的是对照页勾上「慢放三倍」以后的效果：一次只出一两个字，每个字用将近 1.7 秒慢慢显透，一秒十个字上下。

第三回看着舒服，道理在比例上：**淡入的时长要比两个字之间的间隔长得多。** 每秒十几个字，两个字之间隔 0.07 秒；每个字淡入 1.68 秒，屏幕上同时有二十几个字处在不同的浓淡里。眼睛看到的是一道渐变在往前走，看不到哪个字是哪一刻冒出来的。前两回的毛病都是这个比例不够：要么间隔太短、淡入也短，像打字机；要么一下出一整句，淡入再长也藏不住那一下。

最后定下来的参数：

- **底速：每秒 14 个字。**
- **标点后歇一口气。** 句号、问号、叹号后面停大约 2 个字的工夫，逗号、顿号后面停 1 个字，换行停 1.5 个。像人写字有快有慢，不是机器匀速吐字。
- **排队太长就慢慢提速。** 队里超过 300 个字以后，多出来的部分每 16 秒追平一份。公式是：

```
放字速度 = 14 + max(0, 排队字数 − 300) ÷ 16      （字 / 秒）
```

短回答一直按底速走，几千字的长回答也不会拖得太久。不算标点处的停顿，按这个公式算出来是这样（算上停顿，大约再慢一成多）：

| 回答长度 | 模型写字速度 | 模型写完用时 | 屏幕显完用时 | 写完后还要放多久 | 平均显字速度 |
|---|---|---|---|---|---|
| 150 字 | 80 字/秒 | 1.9 秒 | 11 秒 | 9 秒 | 14 字/秒 |
| 400 字 | 80 字/秒 | 5.0 秒 | 28 秒 | 23 秒 | 14 字/秒 |
| 1000 字 | 80 字/秒 | 12.5 秒 | 51 秒 | 38 秒 | 20 字/秒 |
| 2800 字 | 70 字/秒 | 40 秒 | 86 秒 | 46 秒 | 33 字/秒 |

人读中文一秒也就五到八个字，所以显得再慢，也还是比读得快，不会让人干等。嫌慢就把底速调高一点，但淡入时长别跟着缩短。

几个细节：

- **还没放出来的字不进页面。** 这样「自动滚到底」跟的是已经显出来的那一行，闪烁的光标也正好跟在墨的后面。如果把字先全塞进页面、只是藏起来，滚动会一直把你带到一片空白上。
- **跟底要滑，不要跳。** 每长出一行，不要把滚动条一下设到最底，而是每一帧往底部挪剩下距离的一部分（我们用时间常数 0.16 秒的指数追赶），一行字大约半秒滑完。用户手指一碰屏幕就停，把滚动交还给她。
- **每一帧最多按 0.1 秒算。** 手机切到后台、或者弹键盘的时候，浏览器会暂停或放慢动画帧。回来以后如果按真实过去的时间补，会一口气吐出一大截。
- **别把表情劈成两半。** JavaScript 的字符串按 UTF-16 计数，表情和一些生僻字占两个位置。切点正好落在前一半上时，要多带一位，不然会显示成问号块。

---

## 05 · 第四笔：收尾

模型写完的那一刻，大多数聊天页会把「正在输出的那条」换成「历史里的最终消息」。有了队之后，这一刻队里往往还剩着字。如果直接换成普通文本，剩下的字会一下全拍到屏幕上，前面的淡墨就白做了。

- **最终那条消息接着用同一份进度。** 把放字的状态（已经显到第几个字、哪几截还在淡入）放在组件外面。最终消息挂上来时，认出这是同一条，就从原来的位置接着放。正在淡入的那几截，用**负的动画延时**从半路接着演，看不出换过。
- **怎么认出是同一条：** 新文本和旧文本的开头有一段相同（我们取至少 8 个字）。最终文本常常比流式时多做了一点整理，比如去掉首尾空白、删掉内部标签，所以不要求完全一样。
- **用户离开过就别重演。** 如果超过 1.5 秒没在放字（比如她切去了别的页面），回来时把之前到的字都当作已经看过，直接显出来。
- **小字等字显完再出来。** 这个是她试出来的：字还在一截截往外放，底下的时间和 token 已经先冒出来了。原因是消息一落定，页脚就跟着渲染了。修法是放字没结束时先不渲染页脚，放完（`onDone`）再让它轻轻淡入。随回答附带的图片也一样处理。

---

## 06 · 踩过的坑

| 毛病 | 以为是 | 其实是 | 怎么办 |
|---|---|---|---|
| 一大坨字同时到，整段拍到脸上 | 网太卡 | 只淡不排队 | 排进队，按节奏放 |
| 看着像一台很快的打字机 | 速度还是太快 | 两个字的间隔短，淡入也短 | 速度降到每秒 14 个字，淡入拉长到 1.68 秒 |
| 一句一句跳出来，出现那一下很突兀 | 句子太长 | 一整句同时开始淡入 | 回到一个字一个字放，靠长淡入把出现藏住 |
| 每长一行屏幕就往上一跳 | 页面卡了 | 跟底是一下跳过去的 | 每一帧滑一点 |
| 模型越快越不好看 | 淡入时间太短 | 显字速度跟着写字速度走 | 底速加慢慢追 |
| 写完那一下，剩下的字全蹦出来 | 动画没播完 | 换成了最终消息，队丢了 | 最终消息接着用同一份进度 |
| 底下的小字先出来 | 收尾顺序不对 | 消息一落定，页脚就渲染了 | 等放字结束再渲染页脚 |
| 字闪一下又重新淡入 | 动画重播 | 重渲染时改了延时；或者被「切页时临时关动画」的全局开关碰到了 | 延时出生时定死；淡墨不要挂进那种开关 |
| 半截标签被删掉后，字又淡一遍 | 动画重播 | 文本被改写了，不是接着写 | 已经显出来的部分直接落定，不重演 |
| 表情变成问号块 | 字体 | 切点把表情劈成了两半 | 切点落在前一半就多带一位 |
| 改了一行代码，整个页面白屏 | 缓存没刷新 | 新加的变量在声明之前就用了 | 发布之前，在本地把页面真打开一次 |

最后一条是我们自己干过的事。改「小字晚点出来」时，一个变量写在了它的声明前面，整个页面直接打不开了。好在发布前先在本地开了一遍，当场就看见了，她那边没受影响。

---

## 07 · 完整代码

下面是一份能直接用的原生 JavaScript 实现，在线效果那页用的就是它。

```js
// 淡墨：字先进队，一个字一个字慢慢放；每个字用 1.68 秒慢慢显透，前头的还没显透后头的已经跟上。
// 标点后头歇一口气；写完了，队里剩下的字接着放完，然后调 onDone
const INK_PAUSE = { "。": 2.2, "！": 2.2, "？": 2.2, "…": 2.2, "!": 2.2, "?": 2.2,
                    "，": 0.9, "、": 0.9, "；": 0.9, "：": 0.9, ",": 0.9, ";": 0.9, ":": 0.9, "\n": 1.5 };

function createInk(el, opt = {}) {
  const o = { rate: 14, free: 300, tau: 16, fadeMs: 1680, ...opt };
  const st = { target: "", shown: 0, settled: "", chunks: [], acc: 0, tick: 0, raf: 0, ended: false, done: false };
  let settledNode, caret;

  function makeSpan(text, delay) {
    const s = document.createElement("span");
    s.className = "ink";
    s.textContent = text;
    s.style.animationDuration = o.fadeMs + "ms";
    if (delay) s.style.animationDelay = Math.round(delay) + "ms";   // 负数＝从半路接着演
    return s;
  }

  // 第一次挂上；或者收尾时，换到最终那条消息的元素上接着放
  function mount(target) {
    el = target;
    const now = performance.now();
    settledNode = document.createTextNode(st.settled);
    caret = document.createElement("span");
    caret.className = "caret";
    for (const c of st.chunks) c.node = makeSpan(c.text, c.t0 - now);
    el.replaceChildren(settledNode, ...st.chunks.map((c) => c.node), ...(st.done ? [] : [caret]));
  }

  function fold(now) {              // 淡完的字并回普通文本
    while (st.chunks.length && st.chunks[0].t0 + o.fadeMs + 40 <= now) {
      const c = st.chunks.shift();
      st.settled += c.text;
      settledNode.data = st.settled;
      c.node.remove();
    }
  }

  function step() {
    const now = performance.now();
    const dt = Math.min(0.1, (now - st.tick) / 1000);   // 后台回来别一口吐一大截
    st.tick = now;
    if (st.shown < st.target.length) {
      const backlog = st.target.length - st.shown;
      st.acc = Math.min(4, st.acc + (o.rate + Math.max(0, backlog - o.free) / o.tau) * dt);
      if (st.acc >= 1) {
        let end = Math.min(st.target.length, st.shown + Math.floor(st.acc));
        const code = st.target.charCodeAt(end - 1);
        if (code >= 0xD800 && code <= 0xDBFF && end < st.target.length) end++;   // 别把表情劈成两半
        st.acc -= end - st.shown;
        const text = st.target.slice(st.shown, end);
        st.acc -= INK_PAUSE[text[text.length - 1]] || 0;                       // 标点后头歇一口气
        const node = makeSpan(text, 0);
        el.insertBefore(node, caret);
        st.chunks.push({ node, text, t0: now });
        st.shown = end;
        o.onGrow && o.onGrow();
      }
    }
    fold(now);
    if (st.ended && st.shown >= st.target.length && !st.chunks.length) {
      caret.remove();
      st.raf = 0;
      if (!st.done) { st.done = true; o.onDone && o.onDone(); }
      return;
    }
    st.raf = requestAnimationFrame(step);
  }

  function start() {
    if (!st.raf && !st.done) { st.tick = performance.now(); st.raf = requestAnimationFrame(step); }
  }

  mount(el);
  return {
    // 每到一批字，把目前为止的全文递进来
    push(text) {
      const shownText = st.target.slice(0, st.shown);
      if (!text.startsWith(shownText)) {
        // 已经显出来的那截被改了（比如半截标签被删掉）：直接落定，不重演
        let p = 0;
        while (p < st.shown && text.charCodeAt(p) === st.target.charCodeAt(p)) p++;
        st.shown = Math.min(text.length, Math.max(p, st.shown + text.length - st.target.length));
        for (const c of st.chunks) c.node.remove();
        st.chunks = [];
        st.settled = text.slice(0, st.shown);
        settledNode.data = st.settled;
      }
      st.target = text;
      start();
    },
    end() { st.ended = true; start(); },   // 模型写完了；队里的字还会接着放
    mount,
    get busy() { return !st.done; },
    stop() { if (st.raf) cancelAnimationFrame(st.raf); st.raf = 0; },
  };
}
```

光标的样式（可选）：

```css
.caret {
  display: inline-block; width: 3px; height: 1em; margin-left: 3px;
  border-radius: 999px; vertical-align: -0.15em; background: currentColor;
  animation: blink 1.15s ease-in-out infinite;
}
@keyframes blink { 0%, 100% { opacity: 1; } 55% { opacity: .15; } }
```

接到模型的流上（这里用 OpenRouter，别的服务换掉地址和解析就行）：

```js
async function* streamText(messages) {
  const res = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: { "Authorization": "Bearer " + OPENROUTER_KEY, "Content-Type": "application/json" },
    body: JSON.stringify({ model: "anthropic/claude-sonnet-4.6", messages, stream: true }),
  });
  const reader = res.body.getReader();
  const dec = new TextDecoder();
  let buf = "";
  for (;;) {
    const { value, done } = await reader.read();
    if (done) return;
    buf += dec.decode(value, { stream: true });
    let i;
    while ((i = buf.indexOf("\n")) >= 0) {
      const line = buf.slice(0, i).trim();
      buf = buf.slice(i + 1);
      if (!line.startsWith("data:")) continue;
      const data = line.slice(5).trim();
      if (data === "[DONE]") return;
      try {
        const piece = JSON.parse(data).choices?.[0]?.delta?.content;
        if (piece) yield piece;
      } catch {}
    }
  }
}

const bubble = document.querySelector("#reply");
const footer = document.querySelector("#reply-footer");    // 时间、token 那一行
footer.hidden = true;
const ink = createInk(bubble, {
  onGrow: () => window.scrollTo(0, document.body.scrollHeight),   // 换成你自己的「贴底才跟」
  onDone: () => { footer.hidden = false; footer.classList.add("fade-in"); },
});
let full = "";
for await (const piece of streamText(messages)) {
  full += piece;
  ink.push(full);
}
ink.end();
```

「跟底要滑」的一种写法（`box` 是聊天列表的滚动容器；用户手指一碰就停，这部分按你自己的页面接）：

```js
let gliding = false;
function followBottom(box) {
  if (gliding) return;                 // 已经在滑了，目标每一帧现算
  gliding = true;
  let last = performance.now(), pos = box.scrollTop;
  const step = () => {
    const now = performance.now(), dt = Math.min(0.05, (now - last) / 1000);
    last = now;
    const gap = box.scrollHeight - box.clientHeight - pos;
    if (gap <= 1) { gliding = false; return; }
    pos += Math.max(1, gap * (1 - Math.exp(-dt / 0.16)));
    box.scrollTop = pos;
    requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
}
box.addEventListener("touchstart", () => { gliding = false; }, { passive: true });
```

如果你的页面在收尾时会换掉元素（比如把「正在输出」的那条换成历史列表里的正式消息），换完之后调一次 `ink.mount(新的元素)`，再 `ink.push(最终文本)` 就能接着放。

---

## 08 · 用 React 的话

我们自己的页面是 React 写的，接的时候注意这几样：

- **放字的状态别放进 React 的 state。** 进度、队、正在淡入的那几截都放在组件外面，或者放在 `useRef` 里。每放一截字只让这个小组件自己重渲染一次，整个列表不动。
- **一份进度只许一个组件放字。** 收尾交接那一帧，新旧两个组件可能同时在。让后挂上的那个说了算，旧的看到不是自己就停手。
- **最终消息渲染时问一句「还在放吗」。** 还在放，就用同一个淡墨组件接着放，并且先不渲染页脚；放完调一次回调，让父组件重渲染一次，把页脚请出来。
- **动画延时在 `span` 第一次渲染时记下来，**之后的重渲染沿用记下的值。接班的新组件按它自己的挂载时刻重新算（可以是负数）。

---

## 09 · 参数

| 参数 | 我们用的值 | 作用 | 想要的效果 → 怎么调 |
|---|---|---|---|
| `rate` | 14 字/秒 | 底速 | 嫌慢调高，嫌快调低 |
| `free` | 300 字 | 队排到多长才开始提速 | 长回答嫌拖就调小 |
| `tau` | 16 秒 | 追的快慢，越小追得越快 | 长回答嫌拖就调小 |
| `fadeMs` | 1680 毫秒 | 一个字从透明到显全的时间 | 越长越像墨洇开；要 Anthropic 那种利落就调到 200 左右，曲线换成先快后慢 |
| `INK_PAUSE` | 句号 2.2、逗号 0.9、换行 1.5 | 标点后歇几个字的工夫 | 想更匀就调小，想更像写字就调大 |

---

作者·离 · 写于 2026-10-06 晚 · 实测于 iPhone 上的 WebView 和桌面 Chrome。

这一篇是她起的头：她刷到那条帖子，说也想要。第一版做好，她先觉得好看；拿到手机上一试，又嫌字涌得太快。改完再试，她发现字还没显完，底下的小字已经先冒出来了；再改完，她又觉得还是有点快、有一点点卡；改成一句一句地放，她又嫌那一下突兀，指给我们看她最喜欢的是对照页放慢三倍的样子。她提了四回，我们改了四回，这一篇就是这么来的。
