---
title: "我的云端博客"
---

<div class="mt-16 text-center">
  <h1 class="text-4xl font-extrabold tracking-tight text-slate-900 dark:text-slate-100 sm:text-5xl">
    欢迎来到我的云端博客
  </h1>
  <p class="mt-4 text-lg text-slate-600 dark:text-slate-400">
    记录从本地到云端的探索日记、全栈开发与环境配置心得。
  </p>
  <div class="mt-8 flex justify-center gap-4">
    <a href="/posts" class="rounded-full bg-blue-600 px-6 py-3 text-sm font-semibold text-white shadow-sm hover:bg-blue-500">阅读文章</a>
    <a href="/about" class="rounded-full bg-slate-100 px-6 py-3 text-sm font-semibold text-slate-900 shadow-sm ring-1 ring-inset ring-slate-300 hover:bg-slate-200 dark:bg-slate-800 dark:text-slate-100 dark:ring-slate-700 dark:hover:bg-slate-700">关于我</a>
  </div>
</div>

<div class="mt-24">
  {{< cards >}}
    {{< card link="/posts/2026-03-24-local-hugo/" title="🚀 自动化部署流水线" icon="server" subtitle="Hugo + AWS EC2 + GitHub" >}}
    {{< card link="/categories/tech/" title="💻 极客折腾笔记" icon="code" subtitle="Docker / Nginx 代理配置" >}}
    {{< card link="/about/" title="👤 关于作者" icon="user" subtitle="了解更多" >}}
  {{< /cards >}}
</div>