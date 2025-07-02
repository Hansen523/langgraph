---
hide_comments: true
title: LangGraph

<script>
  // 该脚本仅在MkDocs运行，GitHub上不生效
  var hideGitHubVersion = function() {
    document.querySelectorAll('.github-only').forEach(el => el.style.display = 'none');
  };

  // 处理初始加载和后续导航
  document.addEventListener('DOMContentLoaded', hideGitHubVersion);
  document$.subscribe(hideGitHubVersion);
</script>

<p class="mkdocs-only">
  <img class="logo-light" src="static/wordmark_dark.svg" alt="LangGraph Logo" width="80%">
  <img class="logo-dark" src="static/wordmark_light.svg" alt="LangGraph Logo" width="80%">
</p>

<style>
.md-content h1 {
  display: none;
}
.md-header__topic {
  display: none;
}
</style>

{!../README.md!}