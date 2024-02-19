---
title: "My Emacs Dotfile"
author: ["张俊(geekard@qq.com)"]
date: 2023-08-20T00:00:00+08:00
lastmod: 2024-02-19T19:53:26+08:00
tags: ["emacs"]
categories: ["emacs"]
draft: false
series: ["emacs"]
series_order: 1
---

## <span class="section-num">1</span> install {#install}

编译安装最新 Emacs 29:

```bash
brew uninstall emacs-plus@29
brew install emacs-plus@29  --with-no-frame-refocus --with-xwidgets --with-imagemagick --with-poll --with-dragon-icon --with-native-comp --with-poll --HEAD
brew unlink emacs-plus@29 && brew link emacs-plus@29
ln -sf /usr/local/opt/emacs-plus@29/Emacs.app /Applications
```


## <span class="section-num">2</span> init {#init}

`early-init.el` 是 `Emacs` 启动时最开始执行的文件，执行复杂逻辑可能导致启动失败，所以该文件尽量以变量定义为主。

```emacs-lisp
(when (fboundp 'native-compile-async)
  (setenv "LIBRARY_PATH"
          (concat (getenv "LIBRARY_PATH") "/usr/local/opt/gcc/lib/gcc/current:/usr/local/opt/gcc/lib/gcc/current/gcc/x86_64-apple-darwin22/13"))
  (setq native-comp-speed 4)
  (setq native-comp-async-jobs-number 8)
  ;;(setq inhibit-automatic-native-compilation t)
  (setq native-comp-async-report-warnings-errors 'silent)
  )

;; 加载较新的 .el 文件。
(setq-default load-prefer-newer t)
(setq-default lexical-binding t)
(setq lexical-binding t)

;; 在单独文件保存自定义配置，避免污染 ~/.emacs 文件。
(setq custom-file (expand-file-name "~/.emacs.d/custom.el"))
(add-hook 'after-init-hook (lambda () (when (file-exists-p custom-file) (load custom-file))))

(setq my-bin-path '(
		    "/usr/local/opt/findutils/libexec/gnubin"
		    "/Users/zhangjun/go/bin"
		    "/Users/zhangjun/.cargo/bin"
		    ))
;; 设置 Emacs 启动外部程序时（如 lsp server）给它们传入的环境变量。
(mapc (lambda (p)
	(setenv "PATH" (concat p ":" (getenv "PATH"))))
      my-bin-path)

(dolist (env '(("GOPATH" "/Users/zhangjun/go/bin")
	       ("GOPROXY" "https://proxy.golang.org")
	       ("GOPRIVATE" "*.alibaba-inc.com")))
  (setenv (car env) (cadr env)))

;; Emacs 查找外部程序时使用 exec-path 变量而非 PATH 变量，这里单独设置 exec-path。
(let ((paths my-bin-path))
  (dolist (path paths)
    (setq exec-path (cons path exec-path))))
```


## <span class="section-num">3</span> package {#package}

```emacs-lisp
(require 'package)
(setq package-archives '(("elpa" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/gnu/")
			 ("elpa-devel" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/gnu-devel/")
			 ("melpa" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/melpa/")
			 ("nongnu" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/nongnu/")
			 ("nongnu-devel" . "http://mirrors.tuna.tsinghua.edu.cn/elpa/nongnu-devel/")))
(package-initialize)
(when (not package-archive-contents)
  (package-refresh-contents))

(setq use-package-verbose t)
(setq use-package-always-ensure t)
(setq use-package-always-demand t)
(setq use-package-compute-statistics t)

;; 可以升级内置包。
;;(setq package-install-upgrade-built-in t)

;; 安装 vc-use-package 使 use-package 支持使用 :vc 指令从 github 等安装软件包。
(unless (package-installed-p 'vc-use-package)
  (package-vc-install "https://github.com/slotThe/vc-use-package"))
```

-   `M-x use-package-report` : 查看 package 加载时间（按 S 排序）。


## <span class="section-num">4</span> proxy {#proxy}

全局 socks5 代理：

-   Mac 自带的 curl 不支持 socks 代理。
-   [url-retrieve 使用 curl 作为后端实现](https://emacstalk.github.io/post/007/), 这样全局可使用 socks5 代理。
-   需要添加 `--user-agent` 配置, 否则会被 Google 403 Forbidden;

<!--listend-->

```bash
ls -l /usr/local/opt/curl/bin/curl || brew install curl
export PATH="/usr/local/opt/curl/bin:$PATH"
```

```emacs-lisp
;; 将自己安装的 coreutils 添加到 PATH 环境变量和 exec-path 变量中。
(setq my-coreutils-path "/usr/local/opt/curl/bin")
(setenv "PATH" (concat my-coreutils-path ":" (getenv "PATH")))
(setq exec-path (cons my-coreutils-path  exec-path))

(setq my/socks-host "127.0.0.1")
(setq my/socks-port 1080)
(setq my/socks-proxy (format "socks5h://%s:%d" my/socks-host my/socks-port))

(use-package mb-url-http
  :demand
  :vc (:fetcher github :repo dochang/mb-url)
  :init
  (require 'auth-source)
  (let ((credential (auth-source-user-and-password "api.github.com")))
    (setq github-user (car credential)
          github-password (cadr credential))
    (setq github-auth (concat github-user ":" github-password))
    (setq mb-url-http-backend 'mb-url-http-curl
          mb-url-http-curl-program "/usr/local/opt/curl/bin/curl"
          mb-url-http-curl-switches `("-k" "-x" ,my/socks-proxy
                                      "--max-time" "300"
                                      ;;"-u" ,github-auth
                                      "--user-agent" "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.71 Safari/537.36"
                                      ))))

(defun proxy-socks-enable ()
  (interactive)
  (require 'socks)
  (setq url-gateway-method 'socks
        socks-noproxy '("0.0.0.0" "127.0.0.1" "localhost" "10.0.0.0/8" "172.0.0.0/8"
                        "*cn" "*alibaba-inc.com" "*taobao.com" "*antfin-inc.com")
        socks-server `("Default server" ,my/socks-host ,my/socks-port 5))
  (setenv "all_proxy" my/socks-proxy)
  (setenv "ALL_PROXY" my/socks-proxy)
  (setenv "HTTP_PROXY" nil)
  (setenv "HTTPS_PROXY" nil)
  ;;url-retrieve 使用 curl 作为后端实现, 支持全局 socks5 代理。
  (advice-add 'url-http :around 'mb-url-http-around-advice))

(defun proxy-socks-disable ()
  (interactive)
  (require 'socks)
  (setq url-gateway-method 'native
        socks-noproxy nil)
  (setenv "all_proxy" "")
  (setenv "ALL_PROXY" ""))

(proxy-socks-enable)
```


## <span class="section-num">5</span> tuning {#tuning}

性能调优: 参考 [doom core.el](https://github.com/hlissner/doom-emacs/blob/develop/core/core.el)

```emacs-lisp
(use-package epa
  :config
  (setq user-full-name "zhangjun")
  (setq user-mail-address "geekard@qq.com")
  (setq auth-sources '("~/.authinfo.gpg" "~/work/proxylist/hosts_auth"))
  (setq auth-source-cache-expiry 300)
  ;;(setq auth-source-debug t)

  (setq-default
   ;; 缺省使用 email 地址加密。
   epa-file-select-keys nil
   epa-file-encrypt-to user-mail-address
   ;; 使用 minibuffer 输入 GPG 密码。
   epa-pinentry-mode 'loopback
   epa-file-cache-passphrase-for-symmetric-encryption t)
  (require 'epa-file)
  (epa-file-enable))

;; 关闭容易误操作的按键。
(let ((keys '("s-w" "C-z" "<mouse-2>" "s-k" "s-o" "s-t" "s-p" "s-n" "s-," "s-."
	      "s--" "s-0" "s-+" "C-<wheel-down>" "C-<wheel-up>")))
  (dolist (key keys)
    (global-unset-key (kbd key))))

;; macOS 按键调整：s- 表示 Super，S- 表示 Shift, H- 表示 Hyper。
(setq mac-command-modifier 'meta)
;; option 作为 Super 键。
(setq mac-option-modifier 'super)
;; fn 作为 Hyper 键。
(setq ns-function-modifier 'hyper)

;; 提升 io 性能。
(setq process-adaptive-read-buffering nil)
(setq read-process-output-max (* 1024 1024 4))
(setq inhibit-compacting-font-caches t)
(setq-default message-log-max t)
(setq-default ad-redefinition-action 'accept)
(setq bidi-inhibit-bpa t)
(setq bidi-paragraph-direction 'left-to-right)
(setq-default bidi-display-reordering nil)

;; Garbage Collector Magic Hack
;; 提升 vterm buffer、json 文件响应性能。
(use-package gcmh
  :init
  ;;(setq garbage-collection-messages t)
  ;;(setq gcmh-verbose t)
  (setq gcmh-idle-delay 'auto) ;; default is 15s
  (setq gcmh-auto-idle-delay-factor 10)
  (setq gcmh-high-cons-threshold (* 32 1024 1024))
  (gcmh-mode 1)
  (gcmh-set-high-threshold))
```


## <span class="section-num">6</span> face {#face}


### <span class="section-num">6.1</span> ui {#ui}

```emacs-lisp
(when (memq window-system '(mac ns x))
  (tool-bar-mode -1)
  (scroll-bar-mode -1)
  (menu-bar-mode -1)
  (setq use-file-dialog nil)
  (setq use-dialog-box nil))

;; 向下/向上翻另外的窗口。
(global-set-key (kbd "s-v") 'scroll-other-window)
(global-set-key (kbd "C-s-v") 'scroll-other-window-down)

;; 不显示 Title Bar。
;; square corner: undecorated, round corner: undecorated-round
(add-to-list 'default-frame-alist '(undecorated . t))
(add-to-list 'default-frame-alist '(ns-transparent-titlebar . t))
(add-to-list 'default-frame-alist '(selected-frame) 'name nil)
(add-to-list 'default-frame-alist '(ns-appearance . dark))

;; 不在新 frame 打开文件（如 Finder 的 "Open with Emacs") 。
(setq ns-pop-up-frames nil)

;; 复用当前 frame。
(setq display-buffer-reuse-frames t)
(setq frame-resize-pixelwise t)

;; 在 frame 底部显示的窗口列表。
(setq display-buffer-alist
      `((,(rx bos (or
                   "*Apropos*"
                   "*Help*"
                   "*helpful"
                   "*info*"
                   "*Summary*"
                   "*vt"
                   "*lsp-bridge"
                   "*Org"
                   "*Google Translate*"
                   "*eldoc*"
                   " *eglot"
                   "Shell Command Output") (0+ not-newline))
         (display-buffer-below-selected display-buffer-at-bottom)
         (inhibit-same-window . t)
         (window-height . 0.33))))

;; 高亮当前行。
(global-hl-line-mode t)
(setq global-hl-line-sticky-flag t)

;; 显示行号。
(global-display-line-numbers-mode t)

;; 光标和字符宽度一致（如 TAB)
(setq x-stretch-cursor nil)

;; 30: 左右分屏, nil: 上下分屏。
(setq split-width-threshold nil)

;; 像素平滑滚动。
(pixel-scroll-precision-mode t)

;; 启动后最大化显示模式，加 t 参数让 togg-frame-XX 最后运行，这样最大化才生效。
;;(add-hook 'window-setup-hook 'toggle-frame-fullscreen t)
(add-hook 'window-setup-hook 'toggle-frame-maximized t)

;; 刷行显示。
(global-set-key (kbd "<f5>") #'redraw-display)

;; 透明背景。
(defun my/toggle-transparency ()
  (interactive)
  ;; 分别为 frame 获得焦点和失去焦点的不透明度。
  (set-frame-parameter (selected-frame) 'alpha '(90 . 90))
  (add-to-list 'default-frame-alist '(alpha . (90 . 90)))
  (add-to-list 'default-frame-alist '(alpha-background . 90)) ;; Emacs 29
  )

;; 调整窗口大小。
(global-set-key (kbd "s-<left>") 'shrink-window-horizontally)
(global-set-key (kbd "s-<right>") 'enlarge-window-horizontally)
(global-set-key (kbd "s-<down>") 'shrink-window)
(global-set-key (kbd "s-<up>") 'enlarge-window)

;; 切换窗口。
(global-set-key (kbd "s-o") #'other-window)

;; 滚动显示。
(global-set-key (kbd "s-j") (lambda () (interactive) (scroll-up 1)))
(global-set-key (kbd "s-k") (lambda () (interactive) (scroll-down 1)))

;; 内容居中显示。
(use-package olivetti
  :config
  ;; 内容区域宽度，超过后自动折行。
  (setq-default olivetti-body-width 120)
  (add-hook 'org-mode-hook 'olivetti-mode))
;; fill-column 值要小于 olivetti-body-width 才能正常折行。
(setq-default fill-column 100)
```

-   设置 olivetti body 宽度： `C-c | (M-x olivetti-set-width)`
-   olivetti-body-width 和 fill-column 都是 buffer local 变量，需要使用 setq-default 才能在所有 buffer
    中生效。


### <span class="section-num">6.2</span> dashboard {#dashboard}

```emacs-lisp
(use-package dashboard
  :config
  (dashboard-setup-startup-hook)
  (setq-local global-hl-line-mode nil)
  (setq dashboard-banner-logo-title "Happy Hacking & Writing 🎯")
  (setq dashboard-projects-backend #'project-el)
  (setq dashboard-center-content t)
  (setq dashboard-set-heading-icons t)
  (setq dashboard-set-navigator t)
  (setq dashboard-set-file-icons t)
  (setq dashboard-path-max-length 30)
  (setq dashboard-items '((recents . 15) (projects . 8) (agenda . 3))))
```


### <span class="section-num">6.3</span> doom-modeline {#doom-modeline}

doom-modeline 使用 nerd-icons 在 modeline 上显示 icons。nerd-incos 默认使用 Symbols Nerd Fonts Mono，可以使用 `M-x nerd-icons-install-fonts` 安装。

```emacs-lisp
(use-package nerd-icons)
(use-package doom-modeline
  :hook (after-init . doom-modeline-mode)
  :custom
  (doom-modeline-buffer-encoding nil)
  (doom-modeline-env-version t)
  (doom-modeline-env-enable-go nil)
  (doom-modeline-buffer-file-name-style 'truncate-nil) ;; relative-from-project
  (doom-modeline-vcs-max-length 30)
  (doom-modeline-github nil)
  (doom-modeline-time-icon nil)
  :config
  (display-battery-mode 0)
  (column-number-mode t)
  (size-indication-mode t)
  (display-time-mode t)
  (setq display-time-24hr-format t)
  (setq display-time-default-load-average nil)
  (setq display-time-load-average-threshold 20)
  (setq display-time-format "%H:%M ") ;; "%m/%d[%w]%H:%M "
  (setq display-time-day-and-date t)
  (setq indicate-buffer-boundaries (quote left)))

;; 为 vterm-mode 定义简化的 modeline，提升性能。
(doom-modeline-def-modeline 'my-term-modeline
  '(buffer-info) ;; 左侧
  '(misc-info minor-modes input-method)) ;; 右侧
(add-to-list 'doom-modeline-mode-alist '(vterm-mode . my-term-modeline))
```


### <span class="section-num">6.4</span> font {#font}

-   英文字体：[Iosevka Comfy](https://github.com/protesilaos/iosevka-comfy);
-   中文字体：霞鹜文楷屏幕阅读版 [LxgwWenKai-Screen](https://github.com/lxgw/LxgwWenKai-Screen/releases)，屏幕阅读版主要是对字体做了加粗，便于屏幕阅读;
    -   另一种适用于终端显示的中文等宽字体：[Sarasa-Term-SC-Nerd](https://github.com/laishulu/Sarasa-Term-SC-Nerd)
-   英文 Iosevka/Sarasa 字体和中文 LxgwWenKai 字体，按照 1:1 缩放，在偶数字号的情况下可以实现等宽等高;

其他字体：

-   Symbols 字体:  Noto Sans Symbols 和 Noto Sans Symbols2: <https://fonts.google.com/noto>
-   花園明朝：[HanaMinB](http://fonts.jp/hanazono/)
-   Emacs 默认后备字体：[Symbola](https://dn-works.com/ufas/)
    ```emacs-lisp
    (use-package fontaine
      :config
      (setq fontaine-latest-state-file
    	(locate-user-emacs-file "fontaine-latest-state.eld"))

      ;; Iosevka Comfy is my highly customised build of Iosevka with
      ;; monospaced and duospaced (quasi-proportional) variants as well as
      ;; support or no support for ligatures:
      ;; <https://git.sr.ht/~protesilaos/iosevka-comfy>.
      ;;
      ;; Iosevka Comfy            == monospaced, supports ligatures
      ;; Iosevka Comfy Fixed      == monospaced, no ligatures
      ;; Iosevka Comfy Duo        == quasi-proportional, supports ligatures
      ;; Iosevka Comfy Wide       == like Iosevka Comfy, but wider
      ;; Iosevka Comfy Wide Fixed == like Iosevka Comfy Fixed, but wider
      (setq fontaine-presets
    	'((tiny
               :default-family "Iosevka Comfy Wide Fixed"
               :default-height 70)
              (small
               :default-family "Iosevka Comfy Fixed"
               :default-height 90)
              (regular
               :default-height 160) ;; 默认字体 16px, 需要是偶数才能实现等宽等高。
              (medium
               :default-height 110)
              (large
               :default-weight semilight
               :default-height 140
               :bold-weight extrabold)
              (presentation
               :default-weight semilight
               :default-height 170
               :bold-weight extrabold)
              (jumbo
               :default-weight semilight
               :default-height 220
               :bold-weight extrabold)
              (t
               :default-family "Iosevka Comfy"
               :default-weight regular
               :default-height 100
               :fixed-pitch-family nil ; falls back to :default-family
               :fixed-pitch-weight nil ; falls back to :default-weight
               :fixed-pitch-height 1.0
               :fixed-pitch-serif-family nil ; falls back to :default-family
               :fixed-pitch-serif-weight nil ; falls back to :default-weight
               :fixed-pitch-serif-height 1.0
               :variable-pitch-family "Iosevka Comfy Duo"
               :variable-pitch-weight nil
               :variable-pitch-height 1.0
               :bold-family nil ; use whatever the underlying face has
               :bold-weight bold
               :italic-family nil
               :italic-slant italic
               :line-spacing nil)))

      ;; Recover last preset or fall back to desired style from
      ;; `fontaine-presets'.
      (fontaine-set-preset (or (fontaine-restore-latest-preset) 'regular))

      ;; The other side of `fontaine-restore-latest-preset'.
      (add-hook 'kill-emacs-hook #'fontaine-store-latest-preset)

      (define-key global-map (kbd "C-c f") #'fontaine-set-preset)
      (define-key global-map (kbd "C-c F") #'fontaine-set-face-font))

    ;; Persist font configurations while switching themes (doing it with
    ;; my `modus-themes' and `ef-themes' via the hooks they provide).
    (dolist (hook '(modus-themes-after-load-theme-hook ef-themes-post-load-hook))
      (add-hook hook #'fontaine-apply-current-preset))

    (defun my/set-font ()
      (when window-system
        ;; 设置 Emoji 和 Symbol 字体。
        (setq use-default-font-for-symbols nil)
        (set-fontset-font t 'emoji (font-spec :family "Apple Color Emoji")) ;; Noto Color Emoji
        (set-fontset-font t 'symbol (font-spec :family "Symbola")) ;; Apple Symbols, Symbola
        ;; 设置中文字体。
        (let ((font (frame-parameter nil 'font))
    	  (font-spec (font-spec :family "LXGW WenKai Screen")))
          (dolist (charset '(kana han hangul cjk-misc bopomofo))
    	(set-fontset-font font charset font-spec)))))

    ;; emacs 启动后或 fontaine preset 切换时设置字体。
    (add-hook 'after-init-hook 'my/set-font)
    (add-hook 'fontaine-set-preset-hook 'my/set-font)
    ```
-   查看 Emacs 支持的字体名称： `(print (font-family-list))`
-   安装、更新 Icon 字体： `M-x all-the-icons-install-fonts`


### <span class="section-num">6.5</span> theme {#theme}

主题列表：<https://emacsthemes.com/popular/index.html>

```emacs-lisp
(use-package ef-themes
  :demand
  :config
  (mapc #'disable-theme custom-enabled-themes)
  (setq ef-themes-variable-pitch-ui t)
  (setq ef-themes-mixed-fonts t)
  (setq ef-themes-headings
        '(
          ;; level 0 是文档 title，1-8 是文档 header。
          (0 . (variable-pitch light 1.9))
          (1 . (variable-pitch light 1.8))
          (2 . (variable-pitch regular 1.7))
          (3 . (variable-pitch regular 1.6))
          (4 . (variable-pitch regular 1.5))
          (5 . (variable-pitch 1.4))
          (6 . (variable-pitch 1.3))
          (7 . (variable-pitch 1.2))
          (agenda-date . (semilight 1.5))
          (agenda-structure . (variable-pitch light 1.9))
          (t . (variable-pitch 1.1))))
  (setq ef-themes-region '(intense no-extend neutral)))
```

自动切换深浅主题:

-   light: zenburn ef-elea-light ef-spring ef-day doom-one-light
-   dark: sanityinc-tomorrow-eighties zenburn ef-elea-dark ef-night doom-palenight

<!--listend-->

```emacs-lisp
(defun my/load-theme (appearance)
  (interactive)
  (pcase appearance
    ('light (load-theme 'ef-elea-light t))
    ('dark (load-theme 'ef-elea-dark t))))
(add-hook 'ns-system-appearance-change-functions 'my/load-theme)
(add-hook 'after-init-hook (lambda () (my/load-theme ns-system-appearance)))
```


### <span class="section-num">6.6</span> pulsar {#pulsar}

```emacs-lisp
;; 高亮光标移动到的行。
(use-package pulsar
  :config
  (setq pulsar-pulse t)
  (setq pulsar-delay 0.25)
  (setq pulsar-iterations 5)
  (setq pulsar-face 'pulsar-magenta)
  (setq pulsar-highlight-face 'pulsar-yellow)
  (pulsar-global-mode 1)
  (add-hook 'next-error-hook #'pulsar-pulse-line-red))
```


### <span class="section-num">6.7</span> tab-bar {#tab-bar}

```emacs-lisp
(use-package tab-bar
  :custom
  (tab-bar-close-button-show nil)
  (tab-bar-new-button-show nil)
  (tab-bar-history-limit 20)
  (tab-bar-new-tab-choice "*dashboard*")
  (tab-bar-show 1)
  ;; 使用 super + N 来切换 tab。
  (tab-bar-select-tab-modifiers "super")
  :config
  ;; 去掉最左侧的 < 和 >
  (setq tab-bar-format '(tab-bar-format-tabs tab-bar-separator))
  ;; 开启 tar-bar history mode 后才支持 history-back/forward 命令。
  (tab-bar-history-mode t)
  (global-set-key (kbd "s-f") 'tab-bar-history-forward)
  (global-set-key (kbd "s-b") 'tab-bar-history-back)
  (global-set-key (kbd "s-t") 'tab-bar-new-tab)
  (keymap-global-set "s-}" 'tab-bar-switch-to-next-tab)
  (keymap-global-set "s-{" 'tab-bar-switch-to-prev-tab)
  (keymap-global-set "s-w" 'tab-bar-close-tab)
  (global-set-key (kbd "s-0") 'tab-bar-close-tab))
```


### <span class="section-num">6.8</span> sort-tab {#sort-tab}

```emacs-lisp
(use-package sort-tab
  :demand
  :vc (:fetcher github :repo manateelazycat/sort-tab)
  ;; emacs 启动后再启用 sort-tab 防止显示异常。
  :hook (after-init . sort-tab-mode)
  :config
  ;;(sort-tab-mode 1)
  (setq sort-tab-show-index-number t)
  (setq sort-tab-height 40)
  (setq sort-tab-name-max-length 15)
  (global-set-key (kbd "s-n") 'sort-tab-select-next-tab)
  (global-set-key (kbd "s-p") 'sort-tab-select-prev-tab)
  (global-set-key (kbd "s-w") 'sort-tab-close-current-tab)
  ;; (global-set-key (kbd "s-0") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-1") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-2") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-3") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-4") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-5") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-6") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-7") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-8") 'sort-tab-select-visible-tab)
  (global-set-key (kbd "s-9") 'sort-tab-select-visible-tab)
  ;; 设置 tab 颜色，M-x list-colors-display。
  (set-face-foreground 'sort-tab-current-tab-face "peru")
  ;; 不显示背景颜色。
  (set-face-background 'sort-tab-current-tab-face nil)
  ;; 忽略 treemacs-mode 的 buffer显示。
  (setq sort-tab-hide-function '(lambda (buf) (with-current-buffer buf (derived-mode-p 'treemacs-mode)))))
```


## <span class="section-num">7</span> completion {#completion}


### <span class="section-num">7.1</span> vertico {#vertico}

```emacs-lisp
(use-package vertico
  :config
  (require 'vertico-directory)
  (setq vertico-count 20)
  ;; 默认不选中任何候选者，可以避免默认选中文件后当前 buffer 显示该文件内容。
  ;;(setq vertico-preselect 'prompt)
  (vertico-mode 1)
  (define-key vertico-map (kbd "<backspace>") #'vertico-directory-delete-char)
  (define-key vertico-map (kbd "RET") #'vertico-directory-enter))

(use-package emacs
  :init
  ;; minibuffer 不显示光标。
  (setq minibuffer-prompt-properties '(read-only t cursor-intangible t face minibuffer-prompt))
  (add-hook 'minibuffer-setup-hook #'cursor-intangible-mode)
  ;; M-x 只显示当前 mode 支持的命令。
  (setq read-extended-command-predicate #'command-completion-default-include-p)
  ;; 开启 minibuffer 递归编辑。
  (setq enable-recursive-minibuffers t))
```

-   `C-] (command abort-recursive-edit)` 命令可以在任意 buffer 关闭 minibuffer 的编辑模式。


### <span class="section-num">7.2</span> orderless {#orderless}

orderless 补全风格：使用空格分割匹配模式，模式的顺序没有关系，但是 AND 关系。各模式可以使用如下几种类型：

1.  字面量(literally): the component is treated as a literal string that must occur in the candidate.
2.  正则表达式(regexp): the component is treated as a regexp that must match somewhere in the
    candidate.
3.  首字母缩写(initialism): each character of the component should appear as the beginning of a word
    in the candidate, in order. This maps abc to \\&lt;a.\*\\&lt;b.\*\\c.
4.  flex 样式或多个单词前缀：the characters of the component should appear in that order in the
    candidate, but not necessarily consecutively. This maps abc to a.\*b.\*c.

默认情况下，启用字面量和正则表达式匹配。

orderless 的 style dispatchers 机制可以更灵活的定义输入字符串的匹配风格，可以通过变量
`orderless-style-dispatchers` 来定义，默认值为 `orderless-affix-dispatch`, 它使用一种简单的 =前缀或后缀=的字符(串)来表示各种风格：

`!`
: makes the rest of the component match using `orderless-without-literal`, that is, both `!bad
       and bad!` will match strings that `do not contain the substring bad`.

`,`
: uses orderless-initialism.

`=`
: uses orderless-literal.

`~`
: uses orderless-flex.

`%`
: makes the string match ignoring diacritics and similar inflections on characters (it uses
    the function `char-fold-to-regexp` to do this).

! 只能对 `字面量` 匹配取反（orderless-without-literal) ，和其他 dispatch 字符连用时, ! 需要前缀形式，如 `!=.go` 将不匹配含有字面量 .go 的候选者。

```emacs-lisp
(use-package orderless
  :config
  ;; https://github.com/minad/consult/wiki#minads-orderless-configuration
  (defun +orderless--consult-suffix ()
    "Regexp which matches the end of string with Consult tofu support."
    (if (and (boundp 'consult--tofu-char) (boundp 'consult--tofu-range))
        (format "[%c-%c]*$"
                consult--tofu-char
                (+ consult--tofu-char consult--tofu-range -1))
      "$"))

  ;; Recognizes the following patterns:
  ;; * .ext (file extension)
  ;; * regexp$ (regexp matching at end)
  (defun +orderless-consult-dispatch (word _index _total)
    (cond
     ;; Ensure that $ works with Consult commands, which add disambiguation suffixes
     ((string-suffix-p "$" word)
      `(orderless-regexp . ,(concat (substring word 0 -1) (+orderless--consult-suffix))))
     ;; File extensions
     ((and (or minibuffer-completing-file-name
               (derived-mode-p 'eshell-mode))
           (string-match-p "\\`\\.." word))
      `(orderless-regexp . ,(concat "\\." (substring word 1) (+orderless--consult-suffix))))))

  ;; 在 orderless-affix-dispatch 的基础上添加上面支持文件名扩展和正则表达式的 dispatchers 。
  (setq orderless-style-dispatchers (list #'+orderless-consult-dispatch
                                          #'orderless-affix-dispatch))

  ;; 自定义名为 +orderless-with-initialism 的 orderless 风格。
  (orderless-define-completion-style +orderless-with-initialism
    (orderless-matching-styles '(orderless-initialism orderless-literal orderless-regexp)))

  ;; 使用 orderless 和 emacs 原生的 basic 补全风格， 但 orderless 的优先级更高。
  (setq completion-styles '(orderless basic))
  (setq completion-category-defaults nil)
  ;; 进一步设置各 category 使用的补全风格。
  (setq completion-category-overrides
        '(;; buffer name 补全
          (buffer (styles +orderless-with-initialism))
          ;; file path&name 补全, partial-completion 提供了 wildcard 支持。
          (file (styles basic partial-completion))
          (command (styles +orderless-with-initialism))
          (variable (styles +orderless-with-initialism))
          (symbol (styles +orderless-with-initialism))
          ;; eglot will change the completion-category-defaults to flex, BAD!
          ;; https://github.com/minad/corfu/issues/136#issuecomment-1052843656 (eglot (styles . (orderless
          ;; flex))) 使用 M-SPC 来分隔多个筛选条件。
          (eglot (styles +orderless-with-initialism))))
  ;; 使用 SPACE 来分割过滤字符串, SPACE 可以用 \ 转义。
  (setq orderless-component-separator #'orderless-escapable-split-on-space))
```

-   partial-completion 支持 shell wildcards 和部分文件路径，如 /u/s/l for /usr/share/local;
-   已知的 [completion categories](https://gitlab.com/protesilaos/dotfiles/-/blob/master/emacs/.emacs.d/prot-emacs-modules/prot-emacs-completion-common.el#L60);


### <span class="section-num">7.3</span> consult {#consult}

安装 ripgrep 工具命令：

```bash
which rg || brew install ripgrep
```

```emacs-lisp
(use-package consult
  :hook
  (completion-list-mode . consult-preview-at-point-mode)
  :init
  ;; 如果搜索字符少于 3，可以添加后缀#开始搜索，如 #gr#。
  (setq consult-async-min-input 3)
  ;; 从头开始搜索（而非前位置）。
  (setq consult-line-start-from-top t)
  (setq register-preview-function #'consult-register-format)
  (advice-add #'register-preview :override #'consult-register-window)

  ;; 使用 consult 来预览 xref 的引用定义和跳转。
  (setq xref-show-xrefs-function #'consult-xref)
  (setq xref-show-definitions-function #'consult-xref)

  ;; 不搜索 go vendor 目录。
  (setq consult-ripgrep-args
	"rg --null --line-buffered --color=never --max-columns=1000 --path-separator / --smart-case --no-heading --with-filename --line-number --search-zip -g !vendor/")
  :config
  ;; 按 C-l 激活预览，否则 Buffer 列表中有大文件或远程文件时会卡住。
  (setq consult-preview-key "C-l")
  ;; Use minibuffer completion as the UI for completion-at-point. 也可
  ;; 以使用 Corfu 或 Company 等直接在 buffer中 popup 显示补全。
  (setq completion-in-region-function #'consult-completion-in-region)
  ;; 不对 consult-line 结果进行排序（按行号排序）。
  (consult-customize consult-line :prompt "Search: " :sort nil)
  ;; Buffer 列表中不显示的 Buffer 名称。
  (mapcar
   (lambda (pattern) (add-to-list 'consult-buffer-filter pattern))
   '("\\*scratch\\*"
     "\\*Warnings\\*"
     "\\*helpful.*"
     "\\*Help\\*"
     "\\*Org Src.*"
     "Pfuture-Callback.*"
     "\\*epc con"
     "\\*dashboard"
     "\\*Ibuffer"
     "\\*sort-tab"
     "\\*Google Translate\\*"
     "\\*straight-process\\*"
     "\\*Native-compile-Log\\*"
     "[0-9]+.gpg")))

;; consult line 时自动展开 org 内容。
;; https://github.com/minad/consult/issues/563#issuecomment-1186612641
(defun my/org-show-entry (fn &rest args)
  (interactive)
  (when-let ((pos (apply fn args)))
    (when (derived-mode-p 'org-mode)
      (org-fold-show-entry))))
(advice-add 'consult-line :around #'my/org-show-entry)

(global-set-key (kbd "C-c M-x") #'consult-mode-command)
(global-set-key (kbd "C-c i") #'consult-info)
(global-set-key (kbd "C-c m") #'consult-man)
;; 使用 savehist 持久化保存的 minibuffer 历史。
(global-set-key (kbd "C-M-;") #'consult-complex-command)
(global-set-key (kbd "C-x b") #'consult-buffer)
(global-set-key (kbd "C-x 4 b") #'consult-buffer-other-window)
(global-set-key (kbd "C-x 5 b") #'consult-buffer-other-frame)
(global-set-key (kbd "C-x r b") #'consult-bookmark)
(global-set-key (kbd "C-x p b") #'consult-project-buffer)
(global-set-key (kbd "C-'") #'consult-register-store)
(global-set-key (kbd "C-M-'") #'consult-register)
(global-set-key (kbd "M-y") #'consult-yank-pop)
(global-set-key (kbd "M-Y") #'consult-yank-from-kill-ring)
(global-set-key (kbd "M-g e") #'consult-compile-error)
(global-set-key (kbd "M-g f") #'consult-flymake)
(global-set-key (kbd "M-g g") #'consult-goto-line)
(global-set-key (kbd "M-g o") #'consult-outline)
;; consult-buffer 默认已包含 recent file.
;;(global-set-key (kbd "M-g r") #'consult-recent-file)
(global-set-key (kbd "M-g m") #'consult-mark)
(global-set-key (kbd "M-g k") #'consult-global-mark)
(global-set-key (kbd "M-g i") #'consult-imenu)
(global-set-key (kbd "M-g I") #'consult-imenu-multi)
;; 搜索。
(global-set-key (kbd "M-s g") #'consult-grep)
(global-set-key (kbd "M-s G") #'consult-git-grep)
(global-set-key (kbd "M-s r") #'consult-ripgrep)
;; 对文件名使用正则匹配。
(global-set-key (kbd "M-s d") #'consult-find)
(global-set-key (kbd "M-s D") #'consult-locate)
(global-set-key (kbd "M-s l") #'consult-line)
(global-set-key (kbd "M-s M-l") #'consult-line)
;; Search dynamically across multiple buffers. By default search across project buffers. If invoked with a
;; prefix argument search across all buffers.
(global-set-key (kbd "M-s L") #'consult-line-multi)
;; Isearch 集成。
(global-set-key (kbd "M-s e") #'consult-isearch-history)
;;:map isearch-mode-map
(define-key isearch-mode-map (kbd "M-e") #'consult-isearch-history)
(define-key isearch-mode-map (kbd "M-s e") #'consult-isearch-history)
(define-key isearch-mode-map (kbd "M-s l") #'consult-line)
(define-key isearch-mode-map (kbd "M-s L") #'consult-line-multi)
;; Minibuffer 历史。
;;:map minibuffer-local-map)
(define-key minibuffer-local-map (kbd "M-s") #'consult-history)
(define-key minibuffer-local-map (kbd "M-r") #'consult-history)
```

-   `consult-buffer` 显示的 File 列表来源于变量 `recentf-list`;


### <span class="section-num">7.4</span> embark {#embark}

```emacs-lisp
(use-package embark
  :init
  ;; 使用 C-h 来显示 key preifx 绑定。
  (setq prefix-help-command #'embark-prefix-help-command)
  :config
  (setq embark-prompter 'embark-keymap-prompter)
  (global-set-key (kbd "C-;") #'embark-act) ;; embark-dwim
  ;; 描述当前 buffer 可以使用的快捷键。
  (define-key global-map [remap describe-bindings] #'embark-bindings))

;; embark-consult 支持 embark 和 consult 集成，如使用 wgrep 编辑 consult grep/line 的 export 的结果。
(use-package embark-consult
  :after (embark consult)
  :hook  (embark-collect-mode . consult-preview-at-point-mode))

;; 编辑 grep buffers, 可以和 consult-grep 和 embark-export 联合使用。
(use-package wgrep
  :config
  ;; 执行 `wgre-finished-edit` 时自动保存所有 buffer。
  (setq wgrep-auto-save-buffer t)
  (setq wgrep-change-readonly-file t))
```


### <span class="section-num">7.5</span> marginalia {#marginalia}

```emacs-lisp
(use-package marginalia
  :init
  ;; 显示绝对时间。
  (setq marginalia-max-relative-age 0)
  (marginalia-mode))
```


## <span class="section-num">8</span> rime {#rime}

Mac 系统安装 RIME 输入法：

1.  下载鼠鬚管 Squirrel <https://rime.im/download/>，它包含输入法方案。
    -   或者 rime 核心开发者的 Fork: [LEOYoon-Tsaw/squirrel](https://github.com/LEOYoon-Tsaw/squirrel)。
2.  下载 Squirrel 使用的 [librime](https://github.com/rime/librime/releases) （从 Squirrel 的 [CHANGELOG](https://github.com/rime/squirrel/blob/master/CHANGELOG.md) 中获取版本）
3.  重新登录用户，然后就可以使用 `Control-+` 来触发 RIME 输入法了。
4.  在 Mac 的输入法配置程序中将 鼠须管 去掉，只保留 ABC 和搜狗输入法；
5.  部署生效,:
    -   如果修改了 `~/Library/Rime` 下的配置，必须点击鼠须管的 “重新部署” 才能生效。
    -   对于 emacs-rime，如果修改了 `~/Library/Rime` 下的配置，需要执行 `M-x rime-deploy` 生效；

下载 [librime](https://github.com/rime/librime/releases) 库, emacs-rime 使用它与系统的 RIME 交互：

```bash
curl -L -O https://github.com/rime/librime/releases/download/1.8.5/rime-08dd95f-macOS.tar.bz2
bunzip2 rime-08dd95f-macOS.tar.bz2
mkdir ~/.emacs.d/librime
mv rime-08dd95f-macOS/dist ~/.emacs.d/librime
$ ls ~/.emacs.d/librime/dist/
bin/  include/  lib/  share/
rm -rf rime-08dd95f-macOS.tar.bz2
# 如果 MacOS Gatekeeper 阻止第三方软件运行，可以暂时关闭它：
sudo spctl --master-disable
# 后续再开启：sudo spctl --master-enable
```

下载 [iDvel/rime-ice](https://github.com/iDvel/rime-ice.git) 雾凇拼音输入法方案：

```bash
$ mv Rime Rime.bak.20230406
$ cd
$ mkdir ~/Library/Rime
$ git clone https://github.com/iDvel/rime-ice --depth=1
$ cp -r rime-ice/* ~/Library/Rime
# 后续可以 git pull 更新 rime-ice。
```

-   修改 ~/Library/Rime/installation.yaml 文件， 添加 sync_dir: _Users/zhangjun_.emacs.d/sync/rime, 表示将用户数据同步到这个目录下。然后执行 M-x rime-deploy;
-   常见问题：<https://github.com/iDvel/rime-ice/issues/133>

配置个人同步目录（M-x rime-sync）：

```yaml
distribution_code_name: "emacs-rime"
distribution_name: Rime
distribution_version: 1.0.1
install_time: "Thu Apr  6 17:33:36 2023"
# 本机的 ID 标志，默认是一串 UUID
# 生成的文件夹是这个名字，可以改成更好识别的名称
installation_id: "cde8ff26-5e08-466c-bd2d-aac2aeaedb25"
rime_version: 1.8.5
update_time: "Thu Apr  6 21:04:16 2023"
# 同步的路径，默认是当前配置目录下的 `sync/`
sync_dir: /Users/zhangjun/.emacs.d/sync/rime
# 执行 M-x rime-sync 或点击「同步用户数据」后，Rime 会和配置目录下的 *.userdb/ 进行双向更新同步。同步目录
# （/path/RimeSync/MBP-001）下生成的 *.userdb.txt 就是用户词典了，里面都是输入过的内容。
```

RIME 输入法自定义缺省配置中文：

-   注意：对于列表类型的 patch, 必须列出修改后的整个列表值，不支持不分列表。
-   详细参考：<https://github.com/iDvel/rime-ice/blob/main/default.yaml>

<!--listend-->

```yaml
patch:
  schema_list:
    - schema: rime_ice  # 只启用 rime_ice 雾凇拼音输入法方案。
  menu/page_size: 9 # 显示 9 个候选词。
  # 方案选单切换
  switcher/hotkeys:
  - F4
  - "Control+plus" # 按 C-Shit-+ 调出方案选单。
  switcher/fold_options: false # 呼出时不折叠。
  key_binder/bindings:
  - { when: has_menu, accept: equal, send: Page_Down }             # 下一页
  - { when: paging, accept: minus, send: Page_Up }                 # 上一页
  - { when: always, accept: "Control+period", toggle: ascii_mode}   # 中英文切换, Control+equal
  - { when: always, accept: "Control+comma", toggle: ascii_punct} # 中英文标点切换
  #- { when: always, accept: "Control+comma", toggle: full_shape}   # 全角/半角切换
  # emacs_editing， 开启 emacs 绑定惯例，这样可以使用 C-x 来修正拼音。
  # 需要将这些按键加到 rime-translate-keybindings 变量里后才会生效。
  - { When: composing, accept: Control+p, send: Up }
  - { when: composing, accept: Control+n, send: Down }
  - { when: composing, accept: Control+b, send: Left }
  - { when: composing, accept: Control+f, send: Right }
  - { when: composing, accept: Control+a, send: Home }
  - { when: composing, accept: Control+e, send: End }
  - { when: composing, accept: Control+d, send: Delete }
  - { when: composing, accept: Control+k, send: Shift+Delete }
  - { when: composing, accept: Control+h, send: BackSpace }
  - { when: composing, accept: Control+g, send: Escape }
  - { when: composing, accept: Control+bracketleft, send: Escape }
  - { when: composing, accept: Control+y, send: Page_Up }
  - { when: composing, accept: Alt+v, send: Page_Up }
  - { when: composing, accept: Control+v, send: Page_Down }

# 更多按键名称参考: https://github.com/LEOYoon-Tsaw/Rime_collections/blob/master/Rime_description.md
```

模糊音配置：

-   注意：对于列表类型的 patch, 必须列出修改后的整个列表值，不支持不分列表。

<!--listend-->

```yaml
patch:
  # 模糊拼音
  "speller/algebra":
    ### 模糊音
    # 声母
    - derive/^([zcs])h/$1/          # z c s → zh ch sh
    - derive/^([zcs])([^h])/$1h$2/  # zh ch sh → z c s
    #- derive/^l/n/  # n → l
    #- derive/^n/l/  # l → n
    #- derive/^f/h/  # …………
    #- derive/^h/f/  # …………
    # 韵母
    - derive/in/ing/
    - derive/ing/in/

    ### 超级简拼
    - erase/^hm$/ # 响应超级简拼，取消「噷 hm」的独占
    - erase/^m$/  # 响应超级简拼，取消「呣 m」的独占
    - erase/^n$/  # 响应超级简拼，取消「嗯 n」的独占
    - erase/^ng$/ # 响应超级简拼，取消「嗯 ng」的独占
    - abbrev/^([a-z]).+$/$1/   # 超级简拼
    - abbrev/^([zcs]h).+$/$1/  # 超级简拼中，zh ch sh 视为整体（ch'sh → 城市），而不是像这样分开（c'h's'h → 吃好睡好）。

    ### v u 转换，增加对词库中「nue/nve」「qu/qv」等不同注音的支持
    - derive/^([nl])ue$/$1ve/
    - derive/^([nl])ve$/$1ue/
    - derive/^([jqxy])u/$1v/
    - derive/^([jqxy])v/$1u/

    ### 可输入大写字母，做了 xlit 转写是为了适配双拼
    - xlit/āḃçďēḟḡĥīĵḱĺḿńōṕɋŕśťūṽẃẋȳź/ABCDEFGHIJKLMNOPQRSTUVWXYZ/

    ### 自动纠错
    # 有些规则对全拼简拼混输有副作用：如「x'ai 喜爱」被纠错为「xia 下」
    # zh、ch、sh
    - derive/([zcs])h(a|e|i|u|ai|ei|an|en|ou|uo|ua|un|ui|uan|uai|uang|ang|eng|ong)$/h$1$2/  # hzi → zhi
    - derive/([zcs])h([aeiu])$/$1$2h/  # zih → zhi
    # ai
    - derive/^([wghk])ai$/$1ia/  # wia → wai
    # ia
    - derive/([qjx])ia$/$1ai/  # qai → qia
    # ei
    - derive/([wtfghkz])ei$/$1ie/
    # ie
    - derive/([jqx])ie$/$1ei/
    # ao
    - derive/([rtypsdghklzcbnm])ao$/$1oa/
    # ou
    - derive/([ypfm])ou$/$1uo/
    # uo（无）
    # an
    - derive/([wrtypsdfghklzcbnm])an$/$1na/
    # en
    - derive/([wrpsdfghklzcbnm])en$/$1ne/
    # ang
    - derive/([wrtypsdfghklzcbnm])ang$/$1nag/
    - derive/([wrtypsdfghklzcbnm])ang$/$1agn/
    # eng
    - derive/([wrtpsdfghklzcbnm])eng$/$1neg/
    - derive/([wrtpsdfghklzcbnm])eng$/$1egn/
    # ing
    - derive/([qtypdjlxbnm])ing$/$1nig/
    - derive/([qtypdjlxbnm])ing$/$1ign/
    # ong
    - derive/([rtysdghklzcn])ong$/$1nog/
    - derive/([rtysdghklzcn])ong$/$1ogn/
    # iao
    - derive/([qtpdjlxbnm])iao$/$1ioa/
    - derive/([qtpdjlxbnm])iao$/$1oia/
    # ui
    - derive/([rtsghkzc])ui$/$1iu/
    # iu
    - derive/([qjlxnm])iu$/$1ui/
    # ian
    - derive/([qtpdjlxbnm])ian$/$1ain/
    # - derive/([qtpdjlxbnm])ian$/$1ina/ # 和「李娜、蒂娜、缉拿」等常用词有冲突
    # in
    - derive/([qypjlxbnm])in$/$1ni/
    # iang
    - derive/([qjlxn])iang$/$1aing/
    - derive/([qjlxn])iang$/$1inag/
    # ua
    - derive/([g|k|h|zh|sh])ua$/$1au/
    # uai
    - derive/([g|h|k|zh|ch|sh])uai$/$1aui/
    - derive/([g|h|k|zh|ch|sh])uai$/$1uia/
    # uan
    - derive/([qrtysdghjklzxcn])uan$/$1aun/
    # - derive/([qrtysdghjklzxcn])uan$/$1una/ # 和「去哪、露娜」等常用词有冲突
    # un
    - derive/([qrtysdghjklzxc])un$/$1nu/
    # ue
    - derive/([nlyjqx])ue$/$1eu/
    # uang
    - derive/([g|h|k|zh|ch|sh])uang$/$1aung/
    - derive/([g|h|k|zh|ch|sh])uang$/$1uagn/
    - derive/([g|h|k|zh|ch|sh])uang$/$1unag/
    - derive/([g|h|k|zh|ch|sh])uang$/$1augn/
    # iong
    - derive/([jqx])iong$/$1inog/
    - derive/([jqx])iong$/$1oing/
    - derive/([jqx])iong$/$1iogn/
    - derive/([jqx])iong$/$1oign/
    # 其他
    - derive/([rtsdghkzc])o(u|ng)$/$1o/ # do → dou|dong
    - derive/ong$/on/ # lon → long
    - derive/([tl])eng$/$1en/ # ten → teng
    - derive/([qwrtypsdfghjklzxcbnm])([aeio])ng$/$1ng/ # lng → lang、leng、ling、long
```

配置 Emacs:

```emacs-lisp
(use-package rime
  :custom
  (rime-user-data-dir "~/Library/Rime/")
  (rime-librime-root "~/.emacs.d/librime/dist")
  (rime-emacs-module-header-root "/usr/local/opt/emacs-plus@29/include")
  :hook
  (emacs-startup . (lambda () (setq default-input-method "rime")))
  :bind
  (
   :map rime-active-mode-map
   ;; 在已经激活 Rime 候选菜单时，强制在中英文之间切换，直到按回车。
   ("M-j" . 'rime-inline-ascii)
   :map rime-mode-map
   ;; 强制切换到中文模式.
   ("M-j" . 'rime-force-enable)
   ;; 下面这些快捷键需要发送给 rime 来处理, 需要与 default.custom.yaml 文件中的 key_binder/bindings 配置相匹配。
   ;; 中英文切换
   ("C-." . 'rime-send-keybinding)
   ;; 输入法菜单
   ("C-+" . 'rime-send-keybinding)
   ;; 中英文标点切换
   ("C-," . 'rime-send-keybinding)
   ;; 全半角切换
   ;; ("C-," . 'rime-send-keybinding)
   )
  :config
  ;; 在 modline 高亮输入法图标, 可用来快速分辨分中英文输入状态。
  (setq mode-line-mule-info '((:eval (rime-lighter))))
  ;; 将如下快捷键发送给 rime，同时需要在 rime 的 key_binder/bindings 的部分配置才会生效。
  (add-to-list 'rime-translate-keybindings "C-h") ;; 删除拼音字符
  (add-to-list 'rime-translate-keybindings "C-d")
  (add-to-list 'rime-translate-keybindings "C-k")
  (add-to-list 'rime-translate-keybindings "C-a") ;; 跳转到第一个拼音字符
  (add-to-list 'rime-translate-keybindings "C-e") ;; 跳转到最后一个拼音字符
  ;; support shift-l, shift-r, control-l, control-r, 只有当使用系统 RIME 输入法时才有效。
  (setq rime-inline-ascii-trigger 'shift-l)
  ;; 临时英文模式。
  (setq rime-disable-predicates
	'(rime-predicate-ace-window-p
	  rime-predicate-hydra-p
	  rime-predicate-current-uppercase-letter-p
	  ;;rime-predicate-after-alphabet-char-p
	  ;;rime-predicate-prog-in-code-p
	  ))
  (setq rime-show-candidate 'posframe)
  (setq default-input-method "rime")

  (setq rime-posframe-properties
	(list :background-color "#333333"
	      :foreground-color "#dcdccc"
	      :internal-border-width 2))

  ;; 部分 major-mode 关闭 RIME 输入法。
  (defadvice switch-to-buffer (after activate-input-method activate)
    (if (or (string-match "vterm-mode" (symbol-name major-mode))
	    (string-match "dired-mode" (symbol-name major-mode))
	    (string-match "image-mode" (symbol-name major-mode))
	    (string-match "minibuffer-mode" (symbol-name major-mode)))
	(activate-input-method nil)
      (activate-input-method "rime"))))
```

-   使用 [SwitchKey](https://github.com/itsuhane/SwitchKey) 将 Emacs 的默认系统输入法设置为英文，防止搜狗输入法干扰 RIME。
-   后续如果修改 ~/Library/Rime 目录下的内容， 则需要执行命令 `M-x rime-deploy` 命令生效。
-   [雾凇拼音](https://github.com/iDvel/rime-ice) 主页有一些输入用例， 如果你打同样的拼音可以补全相同的中文候选词就证明已经成功用上了雾凇拼音。
-   以词定字：[: 上屏当前词句的第一个字，]: 上屏当前词句的最后一个字。


## <span class="section-num">9</span> org {#org}


### <span class="section-num">9.1</span> org {#org}

    ID: 2A85FD7C-4E6D-4A3B-A991-40E853CB4BBF

```bash
which watchexec || brew install watchexec
```

```emacs-lisp
(use-package org
  :config
  (setq org-ellipsis "..." ;; " ⭍"
        ;; 使用 UTF-8 显示 LaTeX 或 \xxx 特殊字符， M-x org-entities-help 查看所有特殊字符。
        org-pretty-entities t
        org-highlight-latex-and-related '(latex)
        ;; 只显示而不处理和解释 latex 标记，例如 \xxx 或 \being{xxx}, 避免 export pdf 时出错。
        org-export-with-latex 'verbatim
        org-export-with-broken-links t
        ;; export 时不处理 super/subscripting, 等效于 #+OPTIONS: ^:nil 。
        org-export-with-sub-superscripts nil

        ;; 使用 R_{s} 形式的下标（默认是 R_s, 容易与正常内容混淆) 。
        org-use-sub-superscripts nil
        ;; 文件链接使用相对路径, 解决 hugo 等 image 引用的问题。
        org-link-file-path-type 'relative
        org-html-validation-link nil
        ;; 关闭鼠标点击链接。
        org-mouse-1-follows-link nil

        org-hide-emphasis-markers t
        org-hide-block-startup t
        org-hidden-keywords '(title)
	    org-hide-leading-stars t

        org-cycle-separator-lines 2
        org-cycle-level-faces t
        org-n-level-faces 4
        org-indent-indentation-per-level 2
        ;; 内容缩进与对应 headerline 一致。
        org-adapt-indentation t
        org-list-indent-offset 2
	    ;; 代码块不缩进。
        org-src-preserve-indentation t
        org-edit-src-content-indentation 0

        ;; TODO 状态更新记录到 LOGBOOK Drawer 中。
        org-log-into-drawer t
        ;; TODO 状态更新时记录 note.
        org-log-done 'note ;; note, time

        ;; 不在线显示图片，手动点击显示更容易控制大小。
        org-startup-with-inline-images nil
        org-startup-folded 'content
        ;; 如果对 headline 编号则 latext 输出时会导致 toc 缺失，故关闭。
        org-startup-numerated nil
        org-startup-indented t

        ;; 先从 #+ATTR.* 获取宽度，如果没有设置则默认为 300 。
        org-image-actual-width '(300)
        org-cycle-inline-images-display nil

        ;; org-timer 到期时发送声音提示。
        org-clock-sound t)

  ;; 不自动对齐 tag。
  (setq org-tags-column 0)
  (setq org-auto-align-tags nil)
  ;; 显示不可见的编辑。
  (setq org-catch-invisible-edits 'show-and-error)
  (setq org-fold-catch-invisible-edits t)
  (setq org-special-ctrl-a/e t)
  (setq org-insert-heading-respect-content t)
  ;; 支持 ID property 作为 internal link target(默认是 CUSTOM_ID property)
  (setq org-id-link-to-org-use-id t)
  (setq org-M-RET-may-split-line nil)
  (setq org-todo-keywords '((sequence "TODO(t!)" "DOING(d@)" "|" "DONE(D)")
			        (sequence "WAITING(w@/!)" "NEXT(n!/!)" "SOMEDAY(S)" "|" "CANCELLED(c@/!)")))
  (add-hook 'org-mode-hook 'turn-on-auto-fill)
  (add-hook 'org-mode-hook (lambda () (display-line-numbers-mode 0))))

;; 关闭与 pyim 冲突的 C-, 快捷键。
(define-key org-mode-map (kbd "C-,") nil)
(define-key org-mode-map (kbd "C-'") nil)

(global-set-key (kbd "C-c l") #'org-store-link)
(global-set-key (kbd "C-c a") #'org-agenda)
(global-set-key (kbd "C-c c") #'org-capture)
(global-set-key (kbd "C-c b") #'org-switchb)

;; 关闭频繁弹出的 org-element-cache 警告 buffer 。
(setq org-element-use-cache nil)

(use-package org-modern
  :after (org)
  :config
  ;; 各种符号字体：https://github.com/rime/rime-prelude/blob/master/symbols.yaml
  ;;(setq org-modern-star '("◉" "○" "✸" "✿" "✤" "✜" "◆" "▶"))
  (setq org-modern-star '("⚀" "⚁" "⚂" "⚃" "⚄" "⚅"))
  (setq org-modern-block-fringe nil)
  (setq org-modern-block-name
        '((t . t)
          ("src" "»" "«")
          ("SRC" "»" "«")
          ("example" "»–" "–«")
          ("quote" "❝" "❞")))
  ;; 缩放字体时表格边界不对齐，故不美化表格。
  (setq org-modern-table nil)
  (setq org-modern-list '(
			  (?* . "✤")
			  (?+ . "▶")
			  (?- . "◆")))
  (with-eval-after-load 'org (global-org-modern-mode)))

;; 显示转义字符。
(use-package org-appear
  :custom
  (org-appear-autolinks t)
  :hook (org-mode . org-appear-mode))

;; 建立 org 相关目录。
(dolist (dir '("~/docs/org" "~/docs/org/journal"))
  (unless (file-directory-p dir)
    (make-directory dir)))
```


### <span class="section-num">9.2</span> image {#image}

```bash
which pngpaste || brew install pngpaste
which magick || brew install imagemagick
```

-   imagemagick 用于图片分辨率转换, 编译 emacs 时需要指定 `--with-imagemagick` 参数。

拖拽保存图片或 F6 保存剪贴板中图片:

```emacs-lisp
(use-package org-download
  :config
  ;; 保存路径包含 /static/ 时, ox-hugo 在导出时保留后面的目录层次.
  (setq-default org-download-image-dir "./static/images/")
  (setq org-download-method 'directory
        org-download-display-inline-images 'posframe
        org-download-screenshot-method "pngpaste %s"
        org-download-image-attr-list '("#+ATTR_HTML: :width 400 :align center"))
  (add-hook 'dired-mode-hook 'org-download-enable)
  (org-download-enable)
  (global-set-key (kbd "<f6>") #'org-download-screenshot)
  ;; 不添加 #+DOWNLOADED: 注释。
  (setq org-download-annotate-function (lambda (link) (previous-line 1) "")))
```


### <span class="section-num">9.3</span> babel {#babel}

```emacs-lisp
(setq org-confirm-babel-evaluate t)
;; 关闭 C-c C-c 触发 eval code.
(setq org-babel-no-eval-on-ctrl-c-ctrl-c t)
(setq org-src-fontify-natively t)
;; 使用各语言的 Major Mode 来编辑 src block。
(setq org-src-tab-acts-natively t)

;; yaml 从外部的 yaml-mode 切换到内置的 yaml-ts-mode，告诉 babel 使用该内置 mode，
;; 否则编辑 yaml src block 时提示找不到 yaml-mode。
(add-to-list 'org-src-lang-modes '("yaml" . yaml-ts))
(add-to-list 'org-src-lang-modes '("cue" . cue))

(require 'org)
;; org bable 完整支持的语言列表（ob- 开头的文件）：
;; https://git.savannah.gnu.org/cgit/emacs/org-mode.git/tree/lisp 对于官方不支持的语言，可以通过
;; use-pacakge 来安装。
(use-package ob-go)
(use-package ob-rust)
(org-babel-do-load-languages
 'org-babel-load-languages
 '((shell . t)
   (js . t)
   (makefile . t)
   (go . t)
   (emacs-lisp . t)
   (rust . t)
   (python . t)
   (awk . t)
   (css . t)))

(use-package org-contrib)
```


### <span class="section-num">9.4</span> tex {#tex}

在 `~/.emacs.d/templates` 文件中添加一个名为 my-latext 的 tempel 模板，内容如下：

-   如果生成的 pdf 不显示目录，检查文档 #+OPTIONS 参数中的 toc:nil 和 num: 2 是否生效（如在对应行上执行 C-c C-c）。

<!--listend-->

```text
(my-latex "#+DATE: " (format-time-string "%Y-%m-%d %a") n
	  "#+SUBTITLE: 内部资料，注意保密!
#+AUTHOR: 张俊(zj@opsnull.com)
# 中文语言环境（目录等用中文显示）。
#+LANGUAGE: zh-CN
# 不自动输出 titile 和 toc，后续 latext mystyle 中定制输出。
# 但是需要明确通过 num 控制输出的目录级别。
#+OPTIONS: prop:t title:nil num:2 toc:nil ^:nil
#+LATEX_COMPILER: xelatex
#+LATEX_CLASS: ctexart
#+LATEX_HEADER: \\usepackage{/Users/zhangjun/.emacs.d/mystyle}

# 定制 PDF 封面和目录。
#+begin_export latex
% 封面页
\\begin{titlepage}
% 插入标题
\\maketitle
% 插入封面图
%\\ThisCenterWallPaper{0.4}{/path/to/image.png}
% 封面页不编号
\\noindent\\fboxsep=0pt
\\setcounter{page}{0}
\\thispagestyle{empty}
\\end{titlepage}

% 摘要页
\\begin{abstract}
这是一个摘要。
\\end{abstract}

% 目录页
\\newpage
\\tableofcontents
\\newpage
#+end_export
")
```

```emacs-lisp
;; 将安装的 tex 添加到 PATH 环境变量和 exec-path 变量中，后续 Emacs 查询 xelatex 命令使用。
(setq my-tex-path "/Library/TeX/texbin")
(setenv "PATH" (concat my-tex-path ":" (getenv "PATH")))
(setq exec-path (cons my-tex-path  exec-path))

;; engrave-faces 相比 minted 渲染速度更快。
(use-package engrave-faces
  :after ox-latex
  :config
  (require 'engrave-faces-latex)
  (setq org-latex-src-block-backend 'engraved)
  ;; 代码块左侧添加行号。
  (add-to-list 'org-latex-engraved-options '("numbers" . "left"))
  ;; 代码块主题。
  (setq org-latex-engraved-theme 'ef-light))

(defun my/export-pdf (backend)
	    (progn
	      ;;(setq org-export-with-toc nil)
	      (setq org-export-headline-levels 2))
)
(add-hook 'org-export-before-processing-functions #'my/export-pdf)

;; ox- 为对应的导出后端。
;;(use-package ox-reveal) ;; reveal.js
(use-package ox-gfm :defer t) ;; github flavor markdown
(require 'ox-latex)
(with-eval-after-load 'ox-latex
  ;; latex image 的默认宽度, 可以通过 #+ATTR_LATEX :width xx 配置。
  (setq org-latex-image-default-width "0.7\\linewidth")
  ;; 使用 booktabs style 来显示表格，例如支持隔行颜色, 这样 #+ATTR_LATEX: 中不需要添加 :booktabs t。
  (setq org-latex-tables-booktabs t)
  ;; 不保存 LaTeX 日志文件（调试时打开）。
  (setq org-latex-remove-logfiles t)
  ;; 使用支持中文的 xelatex。
  (setq org-latex-pdf-process '("latexmk -xelatex -quiet -shell-escape -f %f"))
  (add-to-list 'org-latex-classes
               '("ctexart"
                 "\\documentclass[lang=cn,11pt,a4paper,table]{ctexart}
                    [NO-DEFAULT-PACKAGES]
                    [PACKAGES]
                    [EXTRA]"
                 ("\\section{%s}" . "\\section*{%s}")
                 ("\\subsection{%s}" . "\\subsection*{%s}")
                 ("\\subsubsection{%s}" . "\\subsubsection*{%s}")
                 ("\\paragraph{%s}" . "\\paragraph*{%s}")
                 ("\\subparagraph{%s}" . "\\subparagraph*{%s}"))))

;; org export html 格式时需要 htmlize.el 包来格式化代码。
(use-package htmlize)
```

自定义样式 mystyle.sty: 对于表格，如果列内容过宽则导出的 pdf 中该列的内容会被截断，可以为表格设置如下属性，将该列 align 设置为 X 来解决： `#+ATTR_LATEX: :environment tabularx :booktabs t :width
\linewidth :align l|l|X`

```latex
\usepackage{wallpaper} % 显示封面图片或页面图片。

\usepackage{color}
\usepackage{xcolor}
\definecolor{winered}{rgb}{0.5,0,0}
\definecolor{lightgrey}{rgb}{0.9,0.9,0.9}
\definecolor{tableheadcolor}{gray}{0.92}
\definecolor{commentcolor}{RGB}{0,100,0}
\definecolor{frenchplum}{RGB}{190,20,83}

% 提示 title
\usepackage[explicit]{titlesec}
\usepackage{titling}
\setlength{\droptitle}{-6em}

% 超链接和书签
\usepackage[colorlinks]{hyperref}
\hypersetup{
  pdfborder={0 0 0},
  colorlinks=true,
  bookmarksopen=true,
  bookmarksnumbered=true, % 书签目录显示编号。
  linkcolor={winered},
  urlcolor={winered},
  filecolor={winered},
  citecolor={winered},
  linktoc=all}

% 安装 noto-cjk 中文字体: git clone https://github.com/googlefonts/noto-cjk.git
\usepackage{fontspec}
\usepackage[utf8x]{inputenc}
\setmainfont{Noto Serif SC}
\setsansfont{Noto Sans SC}[Scale=MatchLowercase]
\setmonofont{Noto Sans Mono CJK SC}[Scale=MatchLowercase]
\setCJKmainfont[BoldFont=Noto Serif SC]{Noto Serif SC}
\setCJKsansfont{Noto Sans SC}
\setCJKmonofont{Noto Sans Mono CJK SC}

\XeTeXlinebreaklocale "zh"
\XeTeXlinebreakskip = 0pt plus 1pt minus 0.1pt

% 添加 email 命令。
\newcommand\email[1]{\href{mailto:#1}{\nolinkurl{#1}}}

% sidewaytable 依赖 rotfloat
\usepackage {rotfloat}

% tabularx 的特殊 align 参数 X 用来对指定列内容自动换行，否则该列内容有可能被截断，
% 解决办法是：在 org-mode 表格前需要加如下属性：
% #+ATTR_LATEX: :environment tabularx :booktabs t :width \linewidth :align l|X
\usepackage{tabularx}
% 美化表格显示效果
\usepackage{booktabs}
% 表格隔行颜色, {1} 开始行, {lightgrep} 奇数行颜色, {} 偶数行颜色(空表示白色)
\rowcolors{1}{lightgrey}{}

\usepackage{parskip}
\setlength{\parskip}{0.5em}
\setlength{\parindent}{0pt}

\usepackage{etoolbox}
\usepackage{calc}

\usepackage[scale=0.85]{geometry}
%\setlength{\headsep}{5pt}

\usepackage{amsthm}
\usepackage{amsmath}
\usepackage{amssymb}
\usepackage{indentfirst}
\usepackage{multicol}
\usepackage{multirow}
\usepackage{linegoal}
\usepackage{graphicx}
\usepackage{fancyvrb}
\usepackage{abstract}
\usepackage{hologo}

\linespread{1}
\graphicspath{{image/}{figure/}{fig/}{img/}{images/}}

\usepackage[font=small,labelfont={bf}]{caption}
\captionsetup[table]{skip=3pt}
\captionsetup[figure]{skip=3pt}

% 下划线、强调和删除线等
\usepackage[normalem]{ulem}
% 列表
\usepackage[shortlabels,inline]{enumitem}
\setlist{nolistsep}
% xeCJK 默认会把黑点用汉字显示，而 Noto 没有这个字体，所以显示效果为一个小点。
% 解决办法是将它设置为 \bullet, 这样显示为实心黑点。Windows 带的楷体、仿宋没有这个问题。
\setlist[itemize]{label=$\bullet$}
% 或者：
%\renewcommand\labelitemi{\ensuremath{\bullet}}
```


### <span class="section-num">9.5</span> slide {#slide}

```emacs-lisp
(use-package org-tree-slide
  :after (org)
  :commands org-tree-slide-mode
  :hook
  ((org-tree-slide-play . (lambda ()
                            (org-fold-hide-block-all)
                            (setq-default x-stretch-cursor -1)
                            (redraw-display)
			        (blink-cursor-mode -1)
                            ;;(org-display-inline-images)
				;;(hl-line-mode -1)
                            ;;(text-scale-increase 1)
                            (read-only-mode 1)))
   (org-tree-slide-stop . (lambda ()
                            (blink-cursor-mode +1)
                            (setq-default x-stretch-cursor t)
                            ;;(text-scale-increase 0)
                            ;;(hl-line-mode 1)
                            (read-only-mode -1))))
  :config
  (setq org-tree-slide-header t)
  (setq org-tree-slide-content-margin-top 0)
  (setq org-tree-slide-heading-emphasis nil)
  (setq org-tree-slide-slide-in-effect t)
  (setq org-tree-slide-activate-message " ")
  (setq org-tree-slide-deactivate-message " ")
  ;;(setq org-tree-slide-modeline-display t)
  ;;(setq org-tree-slide-breadcrumbs " 👉 ")
  (define-key org-mode-map (kbd "<f8>") #'org-tree-slide-mode)
  (define-key org-tree-slide-mode-map (kbd "<f9>") #'org-tree-slide-content)
  (define-key org-tree-slide-mode-map (kbd "<left>") #'org-tree-slide-move-previous-tree)
  (define-key org-tree-slide-mode-map (kbd "<right>") #'org-tree-slide-move-next-tree))
```

-   如果文字居中失效, 可以执行 `M-x redraw-display` 命令来生效。


### <span class="section-num">9.6</span> capture {#capture}

    ID: 8E33B032-60B8-4392-B362-BDFBF4D4F636

org-capture 支持 store-link 和 capture 协议：

1.  store-link：获取浏览器的 URL 和 Title，然后在 kill-ring 中生成一个链接；
2.  capture：根据浏览器复制的内容和指定的 capture-template 名称来创建一个 capture 项目。

打开 MAC “脚本编辑器” ，写入如下内容，保存为 “EmacsClient-Org”，文件格式为 “应用程序”，保存到 /Applications 目录。

```shell
on open location this_URL
    do shell script "/usr/local/bin/emacsclient \"" & this_URL & "\" && open -a Emacs"
end open location
```

-   如果是自编译的 Emmacs, 则 emacsclient 位于 /usr/local/bin 目录下，否则位于 /Applications/Emacs 包中。

编辑 "/Applications/EmacsClient-Org.app/Contents/Info.plist" 文件，在 plist-&gt;dict 部分添加如下内容：

```xml
  <key>CFBundleURLTypes</key>
  <array>
    <dict>
      <key>CFBundleURLName</key>
      <string>org-protocol handler</string>
      <key>CFBundleURLSchemes</key>
      <array>
        <string>org-protocol</string>
      </array>
    </dict>
  </array>
```

然后执行命令：

```shell
xattr -r -d com.apple.quarantine /Applications/EmacsClient-Org.app
```

双击刚才保存到应用程序目录中的 EmacsClient-Org 程序图标，激活 org-proto 协议。

保存浏览器链接：新建一个浏览器书签，Location 内容如下，然后点击该书签，确认 Emacs 有反应，在 Emacs 内按 C-c C-l 自动补全 URL 和 Title.

```javascript
javascript:location.href='org-protocol://store-link?url='+encodeURIComponent(location.href)+'&title='+encodeURIComponent(document.title)
```

需要在浏览器分别建立对应的书签后而且 emacs 以 server 模式运行才可以使用，

```emacs-lisp
(require 'org-protocol)
(require 'org-capture)

(setq org-capture-templates
      '(("c" "Capture" entry (file+headline "~/docs/org/capture.org" "Capture")
         "* %^{Title}\nDate: %U\nSource: %:annotation\nQuote:\n#+BEGIN_QUOTE\n%i\n#+END_QUOTE\n\n"
	 :empty-lines 1)
        ("t" "Todo" entry (file+headline "~/docs/org/todo.org" "Tasks")
         "* TODO %?\n %U %a\n %i"
	 :empty-lines 1)))
```

新建一个浏览器书签，内容如下：

-   template=c 的 c 与上面定义的 capture template 名称相同。
-   具体参考 <https://orgmode.org/manual/The-capture-protocol.html>

<!--listend-->

```javascript
javascript:location.href='org-protocol://capture?template=c'+'&url='+encodeURIComponent(window.location.href)+'&title='+encodeURIComponent(document.title)+'&body='+encodeURIComponent(window.getSelection())
```


### <span class="section-num">9.7</span> journal {#journal}

    ID: 6F1EF9F6-8840-434D-8CE3-2DF10B4D4956

```emacs-lisp
(use-package org-journal
  :commands org-journal-new-entry
  :bind (("C-c j" . org-journal-new-entry))
  :init
  (setq org-journal-prefix-key "C-c j")
  (defun org-journal-save-entry-and-exit()
    (interactive)
    (save-buffer)
    (kill-buffer-and-window))
  :config
  (define-key org-journal-mode-map (kbd "C-c C-e") #'org-journal-save-entry-and-exit)
  (define-key org-journal-mode-map (kbd "C-c C-j") #'org-journal-new-entry)

  (setq org-journal-file-type 'monthly)
  (setq org-journal-dir "~/docs/org/journal")
  (setq org-journal-find-file 'find-file)

  ;; 加密 journal 文件。
  (setq org-journal-enable-encryption t)
  (setq org-journal-encrypt-journal t)
  (defun my-old-carryover (old_carryover)
    (save-excursion
      (let ((matcher (cdr (org-make-tags-matcher org-journal-carryover-items))))
        (dolist (entry (reverse old_carryover))
          (save-restriction
            (narrow-to-region (car entry) (cadr entry))
            (goto-char (point-min))
            (org-scan-tags '(lambda ()
                              (org-set-tags ":carried:"))
                           matcher org--matcher-tags-todo-only))))))
  (setq org-journal-handle-old-carryover 'my-old-carryover)

  ;; journal 文件头。
  (defun org-journal-file-header-func (time)
    "Custom function to create journal header."
    (concat
     (pcase org-journal-file-type
       (`daily "#+TITLE: Daily Journal\n#+STARTUP: showeverything")
       (`weekly "#+TITLE: Weekly Journal\n#+STARTUP: folded")
       (`monthly "#+TITLE: Monthly Journal\n#+STARTUP: folded")
       (`yearly "#+TITLE: Yearly Journal\n#+STARTUP: folded"))))
  (setq org-journal-file-header 'org-journal-file-header-func))

  ;; org-agenda 集成。
  ;; automatically adds the current and all future journal entries to the agenda
  ;;(setq org-journal-enable-agenda-integration t)
  ;; When org-journal-file-pattern has the default value, this would be the regex.
  (setq org-agenda-file-regexp "\\`\\\([^.].*\\.org\\\|[0-9]\\\{8\\\}\\\(\\.gpg\\\)?\\\)\\'")
  (add-to-list 'org-agenda-files org-journal-dir)

  ;; org-capture 集成。
  (defun org-journal-find-location ()
    (org-journal-new-entry t)
    (unless (eq org-journal-file-type 'daily)
      (org-narrow-to-subtree))
    (goto-char (point-max)))
  (setq org-capture-templates
        (cons '("j" "Journal" plain (function org-journal-find-location)
                "** %(format-time-string org-journal-time-format)%^{Title}\n%i%?"
                :jump-to-captured t :immediate-finish t) org-capture-templates))
```

-   不开启 org-journal-enable-agenda-integration, 而是向 org-agenda-files 变量添加日志文件的方式。否则在历史日记被删除的情况下, 可能导致 Dashbard 显示 agenda 时 hang 。

对于 org-mode 文件，可使用 templ 模板在文件开头添加如下内容结间可避免每次打开时提示选择 gpg key:

```text
;; 插入自己的 GnuPG 加密 key。
(my-gpg "# -*- mode:org; epa-file-encrypt-to: (\"geekard@qq.com\") -*-")
```


### <span class="section-num">9.8</span> hugo {#hugo}

```emacs-lisp
(use-package ox-hugo
  :demand
  :config
  (setq org-hugo-base-dir (expand-file-name "~/blog/local.view"))
  (setq org-hugo-section "posts")
  (setq org-hugo-front-matter-format "yaml")
  (setq org-hugo-export-with-section-numbers t)
  (setq org-export-backends '(go md gfm html latex man hugo))
  (setq org-hugo-auto-set-lastmod t))
```


## <span class="section-num">10</span> magit {#magit}

```emacs-lisp
(setq vc-follow-symlinks t)

(use-package magit
  :custom
  ;; 在当前 window 中显示 magit buffer。
  (magit-display-buffer-function #'magit-display-buffer-same-window-except-diff-v1)
  (magit-log-arguments '("-n256" "--graph" "--decorate" "--color"))
  ;; 按照 word 展示 diff。
  (magit-diff-refine-hunk t)
  (magit-clone-default-directory "~/go/src/")
  :config
  ;; diff org-mode 时展开内容。
  (add-hook 'magit-diff-visit-file-hook (lambda() (when (derived-mode-p 'org-mode)(org-fold-show-entry)))))

;; git-link 根据仓库地址、commit 等信息为光标位置生成 URL:
(use-package git-link
  :config
  (setq git-link-use-commit t)

  ;; 重写 gitlab 的 format 字符串，以匹配公司的系统。
  (defun git-link-commit-gitlab (hostname dirname commit)
    (format "https://%s/%s/commit/%s" hostname dirname commit))
  (defun git-link-gitlab (hostname dirname filename branch commit start end)
    (format "https://%s/%s/blob/%s/%s" hostname dirname
	    (or branch commit)
            (concat filename
                    (when start
                      (concat "#"
                              (if end
                                  (format "L%s-%s" start end)
				(format "L%s" start)))))))
)
```

-   `(setq auto-revert-check-vc-info t)` 自动 revert buffer，确保 modeline 上的分支名正确，但是 CPU Profile 显示比较影响性能，故暂不开启。


## <span class="section-num">11</span> coding {#coding}


### <span class="section-num">11.1</span> indent {#indent}

    ANKI_NOTE_HASH: 1adad6280b24b754a001370ce5f340f0

    ANKI_NOTE_ID: 1703514630494

```emacs-lisp
;; 显示缩进。
(use-package highlight-indent-guides
  :custom
  (highlight-indent-guides-method 'column)
  (highlight-indent-guides-responsive 'top)
  (highlight-indent-guides-suppress-auto-error t)
  :config
  (add-hook 'python-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'python-ts-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'yaml-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'yaml-ts-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'js-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'js-ts-mode-hook 'highlight-indent-guides-mode)
  (add-hook 'web-mode-hook 'highlight-indent-guides-mode))

;; c/c++/go-mode indent 风格：总是使用 tab 而非空格.
(setq indent-tabs-mode t)
;; kernel 风格：table 和 offset 都是 tab 缩进，而且都是 8 字符。
;; https://www.kernel.org/doc/html/latest/process/coding-style.html
(setq c-default-style "linux")
(setq tab-width 8)
(setq c-ts-mode-indent-offset 8)
(setq c-ts-common-indent-offset 8)
(setq c-basic-offset 8)
(setq c-electric-pound-behavior 'alignleft)
```


### <span class="section-num">11.2</span> paren {#paren}

```emacs-lisp
;; 彩色括号。
(use-package rainbow-delimiters :hook (prog-mode . rainbow-delimiters-mode))

;; 高亮匹配的括号。
(use-package paren
  :hook (after-init . show-paren-mode)
  :init
  (setq show-paren-when-point-inside-paren t
        show-paren-when-point-in-periphery t)
  (setq show-paren-style 'parenthesis) ;; parenthesis, expression
  (set-face-attribute 'show-paren-match nil :weight 'extra-bold))

;; 智能括号。
(use-package smartparens
  :config
  (require 'smartparens-config)
  (add-hook 'prog-mode-hook #'smartparens-mode)
  ;;(smartparens-global-mode t)
  (show-smartparens-global-mode t))
```


### <span class="section-num">11.3</span> clang {#clang}

安装最新的 llvm 和 clang:

```bash
$ brew install llvm
$ export CPPFLAGS="-I/usr/local/opt/llvm/include"
$ export LDFLAGS="-L/usr/local/opt/llvm/lib/c++ -Wl,-rpath,/usr/local/opt/llvm/lib/c++"
$ export PATH="/usr/local/opt/llvm/bin:$PATH"
$ export LDFLAGS="-L/usr/local/opt/llvm/lib"
```

将 llvm bin 目录添加到 emacs：

```emacs-lisp
(setq my-llvm-path "/usr/local/opt/llvm/bin")
(setenv "PATH" (concat my-llvm-path ":" (getenv "PATH")))
(setq exec-path (cons my-llvm-path  exec-path))
```


### <span class="section-num">11.4</span> python {#python}

使用 brew 为系统安装缺省的 python3, python3/pip3 等会被安装到 /usr/local/bin 目录下, 后续执行 pip3
install 等会被安装到 `/usr/local/lib/python3.11/site-packages` 目录下:

-   pyenv 会导致 shell 启动变慢, 故不使用.

<!--listend-->

```bash
brew install python # 2024.02.19 安装的是  python3.11 版本.
brew install python-tk@3.11  python-gdbm@3.11
```

安装相关依赖包:

```bash
which pylint || brew install pylint
which flake8 || brew install flake8
which pyright || npm update -g pyright
which yapf || pip3 install yapf
which ipython || pip3 install ipython
```

使用 Emacs 内置的 python-mode：

```emacs-lisp
(defun my/python-setup-shell (&rest args)
  (if (executable-find "ipython3")
      (progn
        (setq python-shell-interpreter "ipython3")
        (setq python-shell-interpreter-args "--simple-prompt -i"))
    (progn
      ;; 2024.02.19: brew install python 安装的 python3.11 版本。
      (setq python-shell-interpreter "python3")
      (setq python-interpreter "python3")
      (setq python-shell-interpreter-args "-i"))))

;; 使用 yapf 格式化 python 代码。
(use-package yapfify)

(use-package python
  :init
  (defvar pyright-directory "~/.emacs.d/.cache/lsp/npm/pyright/lib")
  (if (not (file-exists-p pyright-directory))
      (make-directory pyright-directory t))
  ;;(setq python-indent-guess-indent-offset t)
  ;;(setq python-indent-guess-indent-offset-verbose nil)
  ;;(setq python-indent-offset 2)
  ;;(with-eval-after-load 'exec-path-from-shell (exec-path-from-shell-copy-env "PYTHONPATH"))
  :hook
  (python-mode . (lambda ()
                   (my/python-setup-shell)
                   (yapf-mode))))
```

<!--list-separator-->

1.  pyright

    微软不再维护 python-language-server，主力发展 pyright 和 pyglance，所以不再使用 lsp-python-ms 和
    pyls，而使用 lsp-pyright。

    -   python-lanuage-server 的活跃 fork 版本: <https://github.com/python-lsp/python-lsp-server>
    -   lsp-pyright 是 lsp-mode 的 pyright emacs client, 在使用 lsp-bridge 后，只需要安装 pyright npm 包即可，不需要再安装 lsp-pyright.

    pyright <span class="underline">不使用</span> pyenv `.python-version` 指定的 python 版本或 venv 来搜索依赖的 module，而是使用
    `pyrightconfig.json` 文件中配置的 venv 和 venvPath:

    -   venvPath：指定查找 venv 目录的上级目录，可以包含多个 venv 环境；
    -   venv：指定 venvPath 目录下的、使用的虚拟环境名称, pyright 在该 venv 中搜索依赖的 package;

    安装 `pyenv-pyright` 插件来方便的创建和更新 `pyrightconfig.json` 文件：

    ```bash
    git clone https://github.com/alefpereira/pyenv-pyright.git $(pyenv root)/plugins/pyenv-pyright
    ```

    使用方法：

    1.  使用 `pyenv local` 为项目指定 `pyenv virtualenv`;
    2.  使用 `pyenv pyright` 来自动配置 `pyrightconfig.json` 使用上一步指定的 virtualenv；

    pyright 假设源文件位于项目 scr 目录下，但实际可能会在多个其它子目录（甚至嵌套情况）中放置项目源码，即
    `multi-root` 模式（对应于 vscode 中的多 worksapce 目录)，这时可能出现大量 import 错误，可以通过在项目根目录配置
    `pyrightconfig.json` 文件来解决，例如（参考：python module [Import Resolution](https://github.com/microsoft/pyright/blob/main/docs/import-resolution.md)）：

    ```javascript
    {
        "venv": "venv-2.7.18",
        "venvPath": "/Users/zhangjun/.pyenv/versions",
        "verboseOutput": true,
        "reportMissingTypeStubs": false,
        "executionEnvironments": [
            {
                "root": "scripts",
                "extraPaths": [
                    ".",  // scripts 目录下 py 文件导入同级 py 文件的情况
                    "scripts/appinstance_apply"
                ]
            }
        ]
    }
    ```

    executionEnvironments：

    1.  列表中 root 指定各 workspace 的子目录，是有搜索优先级的，所以如果有相同路径前缀的情况，应该从长到短依列出来：根据 python 文件的 from/import 语句来确定root 路径：即从项目根目录（pyrightconfig.json 文件所在目录）开始到文件中导入路径最开始所在目录之间的目录，都应该是 root。
    2.  extraPaths 列表中的路径可以是绝对路径或相对路径（相对于 pyrightconfig.json 文件），用于添加额外的 python
        module 搜索路径；
        -   添加 "." 是因为需要将 scripts 所在的目录也添加到 module 搜索路径，而不仅仅是 scripts 下的子目录；
    3.  官方的实例参考：[Sample Config File](https://github.com/microsoft/pyright/blob/main/docs/configuration.md#sample-config-file) 和 [testState.test.ts](https://github.com/microsoft/pyright/blob/main/packages/pyright-internal/src/tests/testState.test.ts)；

    [pyright 不支持 python 2.x](https://github.com/Microsoft/pyright/issues/21)，如果在上面文件配置 `"pythonVersion": "2.7"` 则会报错。

    修改 pyrightconfig.json 后，需要执行 `M-x lsp-workspace-restart` 来重启 lsp，如果还是有问题，则可以查看
    `*lsp-log*` buffer 的日志。


### <span class="section-num">11.5</span> go {#go}

```bash
which gopls || go install golang.org/x/tools/gopls@latest
```

安装或更新工具：

```emacs-lisp
(defvar go--tools '("golang.org/x/tools/gopls"
                    "golang.org/x/tools/cmd/goimports"
                    "honnef.co/go/tools/cmd/staticcheck"
                    "github.com/go-delve/delve/cmd/dlv"
                    "github.com/zmb3/gogetdoc"
                    "github.com/josharian/impl"
                    "github.com/cweill/gotests/..."
                    "github.com/fatih/gomodifytags"
                    "github.com/davidrjenni/reftools/cmd/fillstruct"))

(defun go-update-tools ()
  (interactive)
  (unless (executable-find "go")
    (user-error "Unable to find `go' in `exec-path'!"))
  (message "Installing go tools...")
  (dolist (pkg go--tools)
    (set-process-sentinel
     (start-process "go-tools" "*Go Tools*" "go" "install" "-v" "-x" (concat pkg "@latest"))
     (lambda (proc _)))))

(use-package go-fill-struct)
(use-package go-impl)
(use-package go-tag
  :init
  (setq go-tag-args (list "-transform" "camelcase"))
  :config
  (define-key go-mode-map (kbd "C-c t a") #'go-tag-add)
  (define-key go-mode-map (kbd "C-c t r") #'go-tag-remove))
(use-package go-playground :commands (go-playground-mode))
```


### <span class="section-num">11.6</span> markdown {#markdown}

```bash
which multimarkdown || brew install multimarkdown
which grip || pip3 install grip
```

multimarkdown 将 markdown 转换为 html 进行 preview，可以结合 xwidget webkit 或 grip 进行实时预览：

```emacs-lisp
(use-package markdown-mode
  :commands (markdown-mode gfm-mode)
  :mode
  (("README\\.md\\'" . gfm-mode)
   ("\\.md\\'" . markdown-mode)
   ("\\.markdown\\'" . markdown-mode))
  :init
  (when (executable-find "multimarkdown")
    (setq markdown-command "multimarkdown"))
  (setq markdown-enable-wiki-links t)
  (setq markdown-italic-underscore t)
  (setq markdown-asymmetric-header t)
  (setq markdown-make-gfm-checkboxes-buttons t)
  (setq markdown-gfm-uppercase-checkbox t)
  (setq markdown-fontify-code-blocks-natively t)
  (setq markdown-gfm-additional-languages "Mermaid")
  (setq markdown-content-type "application/xhtml+xml")
  (setq markdown-css-paths '("https://cdn.jsdelivr.net/npm/github-markdown-css/github-markdown.min.css"
                             "https://cdn.jsdelivr.net/gh/highlightjs/cdn-release/build/styles/github.min.css"))
  (setq markdown-xhtml-header-content "
<meta name='viewport' content='width=device-width, initial-scale=1, shrink-to-fit=no'>
<style>
body {
  box-sizing: border-box;
  max-width: 740px;
  width: 100%;
  margin: 40px auto;
  padding: 0 10px;
}
</style>
<link rel='stylesheet' href='https://cdn.jsdelivr.net/gh/highlightjs/cdn-release/build/styles/default.min.css'>
<script src='https://cdn.jsdelivr.net/gh/highlightjs/cdn-release/build/highlight.min.js'></script>
<script>
document.addEventListener('DOMContentLoaded', () => {
  document.body.classList.add('markdown-body');
  document.querySelectorAll('pre code').forEach((code) => {
    if (code.className != 'mermaid') {
      hljs.highlightBlock(code);
    }
  });
});
</script>
<script src='https://unpkg.com/mermaid@8.4.8/dist/mermaid.min.js'></script>
<script>
mermaid.initialize({
  theme: 'default',  // default, forest, dark, neutral
  startOnLoad: true
});
</script>
"))
```

使用 grip 来预览 markdown 文件，它调用 github markdown API 来渲染文件，从而确保渲染后分隔和 Github
一致。为了避免 API 调用频率限制，可以创建一个空 scop 的 Access Token，然后将 username 和 token 保存到 `~/.authinfo.gpg` 文件中：

```bash
machine api.github.com login geekard@qq.com password YOUR_TOKEN
```

在 Markdown Buffer 中，执行 `M-x grip-mode` 来启用实时预览，然后可以执行如下命令：

-   M-x grip-start-preview
-   M-x grip-stop-preview
-   M-x grip-restart-preview
-   M-x grip-browse-preview 使用浏览器来预览

<!--listend-->

```emacs-lisp
(use-package grip-mode
  :defer
  :after (markdown-mode)
  :config
  (setq grip-preview-use-webkit nil)
  (setq grip-preview-host "127.0.0.1")
  ;; 保存文件时才更新预览。
  (setq grip-update-after-change nil)
  ;; 从 ~/.authinfo 文件获取认证信息。
  (require 'auth-source)
  (let ((credential (auth-source-user-and-password "api.github.com")))
    (setq grip-github-user (car credential)
          grip-github-password (cadr credential)))
  (define-key markdown-mode-command-map (kbd "g") #'grip-mode))
```

为 markdown 文件添加目录：

```emacs-lisp
(use-package markdown-toc
  :after(markdown-mode)
  :config
  (define-key markdown-mode-command-map (kbd "r") #'markdown-toc-generate-or-refresh-toc))
```


### <span class="section-num">11.7</span> shell {#shell}

emacs 使用 `bash-ts-mode` 来编辑 shell 脚本。

安装 bash language server:

```bash
bash-language-server -v &>/dev/null || npm i -g bash-language-server
```

bash language server 使用 shellcheck 做语法检查和静态分析，使用 lsp diagnose 机制来提示错误（不需要再安装flymake/flycheck)。 这里安装 Shell 脚本静态分析工具 ShellCheck, 支持对 shell 进行语法检查和错误诊断:

```bash
shellcheck -V &>/dev/null || brew install shellcheck
```

设置 shell 脚本缩进规则：

```emacs-lisp
(setq sh-basic-offset 4)
(setq sh-indentation 4)
```

其它：

1.  [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)


### <span class="section-num">11.8</span> treesit {#treesit}

```emacs-lisp
;; treesit-auto 自动安装 grammer 和自动将 xx major-mode remap 到对应的
;; xx-ts-mode 上。具体参考变量：treesit-auto-recipe-list
(use-package treesit-auto
  :demand t
  :config
  (setq treesit-auto-install nil)
  (global-treesit-auto-mode))
```

-   执行 M-x treesit-auto-install-all 来安装所有的 treesit modules。


### <span class="section-num">11.9</span> citre {#citre}

安装 GNU global 和 pygments, global 依赖并自动安装 universal-ctags, 通过 pygments 能生成更丰富的 TAG 内容，同时支持reference 搜索。

-   <https://github.com/universal-ctags/citre/blob/master/docs/user-manual/citre-global.md>
-   global 默认使用 brew 安装的 python@3.12 和 pygments, 而不能直接使用 pip install pygments.

<!--listend-->

```bash
brew install global pygments # 提供 global、gtags 命令, gtags 使用 pygments 支持跟多语言

# 在 ~/.bashrc 中添加如下配置：
# 统一的 tags 文件目录
export GTAGSOBJDIRPREFIX=~/.cache/gtags/
mkdir $GTAGSOBJDIRPREFIX
export GTAGSCONF=/usr/local/Cellar/global/*/share/gtags/gtags.conf
# 使用 pygments 支持更多的语言，他噢夹南是支持 reference 搜索。
export GTAGSLABEL=pygments

# 测试项目
cd go/src/github.com/docker/swarm/
# 生成 TAGS 文件
gtags --explain
# reference
global -xr SetPrimary
# definition
global -x SetPrimary
```

citre 是基于 TAGS 文件的代码浏览工具，支持[集成使用 GNU global TAGS 文件](https://github.com/universal-ctags/citre/blob/master/docs/user-manual/citre-global.md)，创建和更新 global tag 文件：

-   M-x citre-global-create-database
-   M-x citre-global-update-database

注意以下两个命令创建的是 ctags 文件（非 global tag 文件），不支持 references，不建议使用：

-   M-x citre-create-tags-file
-   M-x citre-update-tags-file

如果误使用了上面的命令创建 ctags 文件（项目目录中有 .tags 目录），则后续使用 xref-find-references 会
hang，需要删除。

```emacs-lisp
;; GNU Global gtags
(setenv "GTAGSOBJDIRPREFIX" (expand-file-name "~/.cache/gtags/"))
;; brew update 可能会更新 Global 版本，故这里使用 glob 匹配版本号。
(setenv "GTAGSCONF" (car (file-expand-wildcards "/usr/local/Cellar/global/*/share/gtags/gtags.conf")))
(setenv "GTAGSLABEL" "pygments")

(use-package citre
  :init
  ;; 当打开一个文件时，如果可以找到对应 TAGS 文件则自动开启 citre-mode。开启了 citre-mode 后，会自动
  ;; 向 xref-backend-functions hook 添加 citre-xref-backend，从而支持于 xref 和 imenu 的集成。
  (require 'citre-config)
  :config
  ;; 只使用 GNU Global tags。
  (setq citre-completion-backends '(global))
  (setq citre-find-definition-backends '(global))
  (setq citre-find-reference-backends '(global))
  (setq citre-tags-in-buffer-backends  '(global))
  (setq citre-auto-enable-citre-mode-backends '(global))
  ;; citre-config 的逻辑只对 prog-mode 的文件有效。
  (setq citre-auto-enable-citre-mode-modes '(go-ts-mode go-mode python-ts-mode python-mode))
  (setq citre-use-project-root-when-creating-tags t)
  (setq citre-peek-file-content-height 20)
  ;; 上面的 citre-config 会自动开启 citre-mode，然后下面在
  ;; citre-mode-map 中设置的快捷键就会生效。
  (define-key citre-mode-map (kbd "s-.") 'citre-jump)
  (define-key citre-mode-map (kbd "s-,") 'citre-jump-back)
  (define-key citre-mode-map (kbd "s-?") 'citre-peek-reference)
  (define-key citre-mode-map (kbd "s-p") 'citre-peek)
  (define-key citre-peek-keymap (kbd "s-n") 'citre-peek-next-line)
  (define-key citre-peek-keymap (kbd "s-p") 'citre-peek-prev-line)
  (define-key citre-peek-keymap (kbd "s-N") 'citre-peek-next-tag)
  (define-key citre-peek-keymap (kbd "s-P") 'citre-peek-prev-tag)
  (global-set-key (kbd "C-x c u") 'citre-global-update-database)
  ;; 手动添加 citre-xref-backend，-100 表示添加到开头，这样 citre 的结果优先生效。
  (add-hook 'xref-backend-functions #'citre-xref-backend -100))
```


### <span class="section-num">11.10</span> others {#others}

```emacs-lisp
;; xref 的 history 局限于当前窗口（默认全局）。
(setq xref-history-storage 'xref-window-local-history)
;; 快速在其他窗口查看定义。
(global-set-key (kbd "C-M-.") 'xref-find-definitions-other-window)

;; 移动到行或代码的开头、结尾。
(use-package mwim
  :config
  (define-key global-map [remap move-beginning-of-line] #'mwim-beginning-of-code-or-line)
  (define-key global-map [remap move-end-of-line] #'mwim-end-of-code-or-line))

;; 开发文档。
(use-package dash-at-point
  :config
  ;; 可以在搜索输入中指定 docset 名称，例如： spf13/viper: getstring
  (global-set-key (kbd "C-c d .") #'dash-at-point)
  ;; 提示选择 docset;
  (global-set-key (kbd "C-c d d") #'dash-at-point-with-docset)
  ;; 扩展提示可选的 docset 列表， 名称必须与 dash 中定义的一致。
  (add-to-list 'dash-at-point-docsets "go")
  (add-to-list 'dash-at-point-docsets "viper")
  (add-to-list 'dash-at-point-docsets "cobra")
  (add-to-list 'dash-at-point-docsets "pflag")
  (add-to-list 'dash-at-point-docsets "k8s/api")
  (add-to-list 'dash-at-point-docsets "k8s/apimachineary")
  (add-to-list 'dash-at-point-docsets "k8s/client-go")
  (add-to-list 'dash-at-point-docsets "klog")
  (add-to-list 'dash-at-point-docsets "k8s/controller-runtime")
  (add-to-list 'dash-at-point-docsets "k8s/componet-base")
  (add-to-list 'dash-at-point-docsets "k8s.io/kubernetes"))

(use-package expand-region
  :config
  (global-set-key (kbd "C-=") #'er/expand-region))
```


### <span class="section-num">11.11</span> chatgpt-shell {#chatgpt-shell}

在 ~/.authinfo.gpg 文件中添加 api.openai.com 的 key，然后使用本地 socks5h 代理访问 API。

```emacs-lisp
(use-package shell-maker)
(use-package ob-chatgpt-shell :defer t)
(use-package ob-dall-e-shell :defer t)
(use-package chatgpt-shell
  :requires shell-maker
  :defer t
  :config
  (setq chatgpt-shell-openai-key
        (auth-source-pick-first-password :host "jpaia.openai.azure.com"))
  (setq chatgpt-shell-chatgpt-streaming t)
  (setq chatgpt-shell-model-version "gpt-4-32k") ;; gpt-3.5-turbo gpt-4-32k
  (setq chatgpt-shell-request-timeout 300)
  (setq chatgpt-shell-insert-queries-inline t)
  (require 'ob-chatgpt-shell)
  (ob-chatgpt-shell-setup)
  (require 'ob-dall-e-shell)
  (ob-dall-e-shell-setup)
  ;;(setq chatgpt-shell-api-url-base "http://127.0.0.1:1090")
  (setq chatgpt-shell-api-url-path  "/openai/deployments/gpt-4-32k/chat/completions?api-version=2023-03-15-preview")
	;;"/openai/deployments/gpt-4/chat/completions?api-version=2023-03-15-preview")
  (setq chatgpt-shell-api-url-base "https://jpaia.openai.azure.com/")
  ;; azure 使用 api-key 而非 openai 的 Authorization: Bearer 认证头部。
  (setq chatgpt-shell-auth-header
	(lambda ()
	  (format "api-key: %s" (auth-source-pick-first-password :host "jpaia.openai.azure.com")))))
```


### <span class="section-num">11.12</span> flymake {#flymake}

eglot 使用 Emacs 内置的 flymake 而非 flycheck 来接收和显示 LSP Server 发送的 publishDiagnostics 事件。
flymake 默认在三种情况下检查 buffer 错误：

1.  执行 `M-x flymake-start` 命令；
2.  flymake-no-changes-timeout 时间以后，默认为 0.5， 设置为 nil 后表示无限长。
3.  保存 buffer 时。

将 flymake-no-changes-timeout 设置为 nil 后，eglot 不会显示实时的诊断消息，而是当保存 buffer 时，经过 eglot-send-changes-idle-time 时间后才显示 LSP 诊断消息，这样可以避免显示无意义的错误。

-   <https://github.com/joaotavora/eglot/commit/2b87b06d9ef15e7c39d87fd5a4375b6deaa7e322>

<!--listend-->

```emacs-lisp
(use-package flymake
  :config
  (setq flymake-no-changes-timeout nil)
  (global-set-key (kbd "C-s-l") #'consult-flymake)
  (define-key flymake-mode-map (kbd "C-s-n") #'flymake-goto-next-error)
  (define-key flymake-mode-map (kbd "C-s-p") #'flymake-goto-prev-error))
```

-   M-x flymake-show-buffer-diagnostics
-   M-x flymake-show-project-diagnostics


### <span class="section-num">11.13</span> eldoc {#eldoc}

eldoc 是 echo area 显示当前 symbol 信息，如函数签名或参数类型。global-eldoc-mode 变量默认为 t，则表示 eldoc 默认在所有 major mode 均开启。

```emacs-lisp
(use-package eldoc
  :config
  ;; 打开或关闭 *eldoc* 函数帮助或 hover buffer。
  (global-set-key (kbd "M-`")
                  (
                   lambda()
                   (interactive)
                   (if (get-buffer-window "*eldoc*")
                       (delete-window (get-buffer-window "*eldoc*"))
                     (display-buffer "*eldoc*")))))
```

-   M-x eldoc 或 C-h .(eldoc-doc-buffer): 在独立的 buffer **eldoc** 中显示 eldoc 文档；


### <span class="section-num">11.14</span> corfu {#corfu}

A minimal ui for completion-in-region。corfu 与 orderless 的匹配性更好，比如可以对候选词使用
orderless 的过滤方式。但是 company-mode 与 orderless 的匹配性不好，不能使用空格，模糊匹配等特性。

```emacs-lisp
(use-package corfu
  :init
  (global-corfu-mode 1) ;; 全局模式，eshell 等也会生效。
  (corfu-popupinfo-mode 1) ;;  显示候选者文档。
  :custom
  (corfu-cycle t)                ;; Enable cycling for `corfu-next/previous'
  (corfu-auto t)                 ;; Enable auto completion
  (corfu-separator ?\s)          ;; Orderless field separator
  (corfu-preselect 'prompt)      ;; Preselect the prompt
  (corfu-scroll-margin 5)        ;; Use scroll margin
  :config
  ;; Enable `corfu-history-mode' to sort candidates by their history position.
  (savehist-mode 1)
  (add-to-list 'savehist-additional-variables 'corfu-history)

  (defun corfu-enable-always-in-minibuffer ()
    (setq-local corfu-auto nil)
    (corfu-mode 1))
  (add-hook 'minibuffer-setup-hook #'corfu-enable-always-in-minibuffer 1)

  ;; eshell 使用 pcomplete 来自动补全，eshell 自动补全。
  (add-hook 'eshell-mode-hook
            (lambda ()
              (setq-local corfu-auto nil)
              (corfu-mode))))

(use-package emacs
  :init
  ;; 总是在弹出菜单中显示候选者。 TAB cycle if there are only few candidates
  (setq completion-cycle-threshold nil)
  ;; 使用 TAB 来 indentation+completion(completion-at-point 默认是 M-TAB) 。
  (setq tab-always-indent 'complete))

(use-package kind-icon
  :after corfu
  :demand
  :custom
  (kind-icon-default-face 'corfu-default)
  :config
  (add-to-list 'corfu-margin-formatters #'kind-icon-margin-formatter))
```


### <span class="section-num">11.15</span> cape {#cape}

```emacs-lisp
;; cape 补全融合
(use-package cape
  :init
  ;; completion-at-point 使用的函数列表，注意顺序。
  (add-to-list 'completion-at-point-functions #'cape-file)
  ;;(add-to-list 'completion-at-point-functions #'cape-dabbrev)
  (add-to-list 'completion-at-point-functions #'cape-elisp-block)
  ;;(add-to-list 'completion-at-point-functions #'cape-symbol)
  ;;(add-to-list 'completion-at-point-functions #'cape-keyword)
  ;;(add-to-list 'completion-at-point-functions #'cape-history)
  ;;(add-to-list 'completion-at-point-functions #'cape-tex)
  ;;(add-to-list 'completion-at-point-functions #'cape-sgml)
  ;;(add-to-list 'completion-at-point-functions #'cape-rfc1345)
  ;;(add-to-list 'completion-at-point-functions #'cape-abbrev)
  ;;(add-to-list 'completion-at-point-functions #'cape-dict)
  ;;(add-to-list 'completion-at-point-functions #'cape-line)
  :config
  (setq dabbrev-check-other-buffers nil
        dabbrev-check-all-buffers nil
        cape-dabbrev-min-length 3)
  ;; 前缀长度达到 3 时才调用 CAPF，避免频繁调用自动补全。
  (cape-wrap-prefix-length #'cape-dabbrev 3))
```


### <span class="section-num">11.16</span> tempel {#tempel}

```emacs-lisp
(use-package tempel
  :bind (("M-+" . tempel-complete)
         ("M-*" . tempel-insert))
  :init
  (defun tempel-setup-capf ()
    (setq-local completion-at-point-functions
                (cons #'tempel-expand
                      completion-at-point-functions)))
  (add-hook 'conf-mode-hook 'tempel-setup-capf)
  (add-hook 'prog-mode-hook 'tempel-setup-capf)
  (add-hook 'text-mode-hook 'tempel-setup-capf)
  ;; 确保 tempel-setup-capf 位于 eglot-managed-mode-hook 前，这样 corfu 才会显示 tempel 的自动补全。
  ;; https://github.com/minad/tempel/issues/103#issuecomment-1543510550
  (add-hook #'eglot-managed-mode-hook 'tempel-setup-capf))

(use-package tempel-collection)
```

-   可以在变量 tempel-path 定义的文件中 `~/.emacs.d/templates` 添加自定义模板。


### <span class="section-num">11.17</span> eglot {#eglot}

elgot 使用 Emacs 内置的 flymake（而非 flycheck）、xref、eldoc、project。

前面打开 package-install-upgrade-built-in 后，就可以升级内置的 eglot了。eglot 是通过向
flymake-diagnostic-functions hook 添加'eglot-flymake-backend 来实现诊断的。

eglot 启动后，将 xref-backend-functions 设置为 eglot-xref-backend，而忽略已注册的其它 backend，解决办法是使用 .dir-local 文件关闭 eglot mode。

-   查看 eglot 报错：在开启 eglot 的 buffer 执行命令 `M-x eglot-stderr-buffer`

<!--listend-->

```emacs-lisp
(use-package eglot
  :demand
  :bind (:map eglot-mode-map
	      ("C-c C-a" . eglot-code-actions)
	      ;; 如果 buffer 出现错误的诊断消息，可以执行 flymake-start 命令来重新触发诊断。
	      ("C-c C-c" . flymake-start)
	      ("C-c C-d" . eldoc)
	      ("C-c C-f" . eglot-format-buffer)
	      ("C-c C-r" . eglot-rename))
  :config
  ;; 将 eglot-events-buffer-size 设置为 0 后将关闭显示 *EGLOT event* bufer，不便于调试问题。也不能设
  ;; 置的太大，否则可能影响性能。
  (setq eglot-events-buffer-size (* 1024 1024 1))
  ;; 将 flymake-no-changes-timeout 设置为 nil 后，eglot 在保存 buffer 内容后，经过 idle time 才会显
  ;; 示 LSP 发送的诊断消息。
  (setq eglot-send-changes-idle-time 0.3)

  ;; Shutdown server when last managed buffer is killed
  (customize-set-variable 'eglot-autoshutdown t)
  (customize-set-variable 'eglot-connect-timeout 60)   ;; default 30s

  ;; 不能给所有 prog-mode 都开启 eglot，否则当它没有 language server时，eglot 报错。由于
  ;; treesit-auto已经对 major-mode 做了 remap ，这里需要对 xx-ts-mode-hook 添加 hook，而不是以前的
  ;; xx-mode-hook。如果代码项目没有 .git 目录，则打开文件时可能会卡主。
  (add-hook 'c-ts-mode-hook #'eglot-ensure)
  (add-hook 'go-ts-mode-hook #'eglot-ensure)
  (add-hook 'bash-ts-mode-hook #'eglot-ensure)
  (add-hook 'python-ts-mode-hook #'eglot-ensure)
  (add-hook 'rust-ts-mode-hook #'eglot-ensure)

  ;; 忽略一些用不到，耗性能的能力。
  (setq eglot-ignored-server-capabilities
	'(
	  ;;:hoverProvider ;; 显示光标位置信息。
	  ;;:documentHighlightProvider ;; 高亮当前 symbol。
	  :inlayHintProvider ;; 显示 inlay hint 提示。
	  ))

  ;; 加强高亮的 symbol 效果。
  ;; (set-face-attribute 'eglot-highlight-symbol-face nil
  ;;                     :background "#b3d7ff")

  ;; ;; 在 eldoc bufer 中只显示帮助文档。
  (defun my/eglot-managed-mode-initialize ()
    ;; ;; 不显示 flymake 错误和函数签名，放置后续的 eldoc buffer 内容来回变。
    ;; (setq-local
    ;;  eldoc-documentation-functions
    ;;  (list
    ;;   ;; 关闭自动在 eldoc 显示 flymake 的错误， 这样 eldoc 只显示函数签名或文档，后续 flymake 的错误单独在
    ;;   ;; echo area 显示。
    ;;   ;;#'flymake-eldoc-function
    ;;   #'eglot-signature-eldoc-function ;; 关闭自动在 eldoc 自动显示函数签名，使用 M-x eldoc 手动显示函数帮助。
    ;;   #'eglot-hover-eldoc-function))

    ;; 在单独的 buffer 中显示 eldoc 而非 echo area。
    (setq-local
     eldoc-display-functions
     (list
      #'eldoc-display-in-echo-area
      #'eldoc-display-in-buffer))
  (add-hook 'eglot-managed-mode-hook #'my/eglot-managed-mode-initialize))

  ;; t: true, false: :json-false 而不是 nil。
  (setq-default eglot-workspace-configuration
		'((:gopls .
			  ((staticcheck . t)
			   (usePlaceholders . :json-false)
			   (matcher . "CaseSensitive"))))))

;; 具体参数列表参考：https://rust-analyzer.github.io/manual.html#configuration
(add-to-list 'eglot-server-programs
             '((rust-ts-mode rust-mode) .
               ("rust-analyzer" :initializationOptions
                (
		      :checkOnSave :json-false
	              ;;:diagnostics (:enable :json-false)
		      ;;:cachePriming (:enable :json-false)
		      ;; https://esp-rs.github.io/book/tooling/visual-studio-code.html#using-rust-analyzer-with-no_std
		      :check (:allTargets  :json-false)
		      :procMacro ( :attributes (:enable t)
		       		   :enable :json-false)
                   :cargo ( :buildScripts (:enable :json-false)
                            :features "all"
		                :cfgs (:tokio_unstable "")
				:autoreload :json-false)
	              :diagnostics (:disabled ["unresolved-proc-macro"
                                           "unresolved-macro-call"]))
	        )))

;; 由于 major-mode 开启 eglot-ensure 后，eglot 将 xref-backend-functions 设置为 eglot-xref-backend，
;; 而忽略已注册的其它 backend。这里定义一个一键切换函数，在 lsp 失效的情况下，可以手动关闭当前
;; major-mode 的 eglot，从而让 xref-backend-functions 恢复为以前的值，如 dump-jump-xref-active。
(defun my/toggle-eglot ()
  (interactive)
  (let ((current-mode major-mode)
        (hook (intern (concat (symbol-name major-mode) "-hook"))))
    (if (bound-and-true-p eglot--managed-mode)
        (progn
          (eglot-shutdown-all)
          (remove-hook hook 'eglot-ensure))
      (progn
        (add-hook hook 'eglot-ensure)
        (eglot-ensure)))))
(global-set-key (kbd "s-`") 'my/toggle-eglot)
```

-   更新内置的 elgot： M-x eglot-upgrade-eglot。
-   eldoc，eldoc-doc-buffer： C-h-.

consult-eglot 提供 `consult-eglot-symbols` 函数，可以选择 workspace 中的 symbol：

```emacs-lisp
(use-package consult-eglot
  :after (eglot consult))
```


### <span class="section-num">11.18</span> emacs-lsp-booster {#emacs-lsp-booster}

下载 [emacs-lsp-booster](https://github.com/blahgeek/emacs-lsp-booster) 可执行程序：

```bash
which emacs-lsp-booster || wget https://github.com/blahgeek/emacs-lsp-booster/releases/download/v0.2.0/emacs-lsp-booster_v0.2.0_x86_64-apple-darwin.zip
```

使用 emacs-lsp-booster 来加速 eglot 的响应性能：

```emacs-lisp
(use-package eglot-booster
  :vc (:fetcher github :repo jdtsmith/eglot-booster)
	:after eglot
	:config	(eglot-booster-mode))
```


### <span class="section-num">11.19</span> rust {#rust}

安装 rust 工具链，这里使用 rustup 来管理工具链和版本：

-   rust-analyzer [官方手册](https://rust-analyzer.github.io/manual.html#rust-analyzer-language-server-binary)。

<!--listend-->

```bash
# 清理旧环境
mv ~/.cargo{,.bak}
brew uninstall rust rust-analyzer

brew install rustup-init
echo 'export PATH=$HOME/.cargo/bin:$PATH' >>~/.bashrc

rustup-init   # 下载 rust stable 工具链
rustup component add rust-analyzer # 安装 rust lsp server
rustup component add clippy  # rust lints
rustup component add rust-src
rustup component add rust-docs # 添加 rust 标准库文档
rustup toolchain list   # 查看安装的工具链
```

如果要浏览 rust github 官方库，则需要进一步操作：

```bash
rustup toolchain install nightly # rust 仓库依赖 nightly 版本工具链
rustup default nightly           # 将工具链切换到 nightly 版本
rustup component add rust-analyzer # 安装 nightly 版本的 rust lsp server
rustup component add rust-docs # 添加 rust 标准库文档

# cd 到 rust github 仓库
/Users/zhangjun/go/src/github.com/rust-lang/rust
git submodule init
git submodule update library/*  # clone 依赖的库
```

查看文档：

```bash
rustup component add rust-docs # 添加 rust 标准库文档
rustup doc # 查看标准库文档
rustup doc topic # 查看某个 topic 的帮助文档，如 core，fn，std:char 等。
cargo doc --open # 查看当前项目和依赖的文档
```

使用当前项目的文档：

-   先切换到 Rust 项目，然后执行 M-x cargo-process-doc-open， 然后使用 M-x xwidget-webkit-browse-url
    来打开 URL 即可。

<!--listend-->

```emacs-lisp
;; 将 brew rustup-init 安装的目录添加到 PATH 和 emacs exec-path 中。
(setq my-cargo-path "/Users/zhangjun/.cargo/bin")
(setenv "PATH" (concat my-cargo-path ":" (getenv "PATH")))
(setq exec-path (cons my-cargo-path  exec-path))
;; https://github.com/mozilla/sccache?tab=readme-ov-file
;; cargo install sccache --locked
(setenv "RUSTC_WRAPPER" "/Users/zhangjun/.cargo/bin/sccache")

;; https://github.com/jwiegley/dot-emacs/blob/master/init.org#rust-mode
(use-package rust-mode
  :mode "\\.rs\\'"
  :bind (:map rust-mode-map
              ("M-n" . flymake-goto-next-error)
              ("M-p" . flymake-goto-prev-error)
              ("C-c C-c v" . (lambda ()
                               (interactive)
                               (shell-command "rustdocs std"))))
  :custom
  (rust-format-on-save t))
```


## <span class="section-num">12</span> neotree {#neotree}

```emacs-lisp
(use-package neotree
  :config
  (setq neo-smart-open t) ;; 自动跳转到当前打开的文件。
  (setq neo-theme 'nerd) ;; nerd 更简洁。
  (setq neo-vc-integration '(face))
  (setq neo-window-width 30)
  (setq neo-window-fixed-size nil)
  (setq neo-theme (if (display-graphic-p) 'icons 'arrow))
  :bind
      (:map global-map
	      ("s-0" . neotree-toggle)))
(add-hook 'neotree-mode-hook (lambda () (display-line-numbers-mode 0)))

;; 根据项目切换 neotree root 目录。
(defun my-neotree-project-dir-toggle ()
  "Open NeoTree using the project root, using projectile, find-file-in-project,
or the current buffer directory."
  (interactive)
  (require 'neotree)
  (let* ((filepath (buffer-file-name))
         (project-dir
          (with-demoted-errors "neotree-project-dir-toggle error: %S"
              (cond
               ((featurep 'projectile)
                (projectile-project-root))
               ((featurep 'find-file-in-project)
                (ffip-project-root))
               (t ;; Fall back to version control root.
                (if filepath
                    (vc-call-backend
                     (vc-responsible-backend filepath) 'root filepath)
                  nil)))))
         (neo-smart-open t))

    (if (and (fboundp 'neo-global--window-exists-p)
             (neo-global--window-exists-p))
        (neotree-hide)
      (neotree-show)
      (when project-dir
        (neotree-dir project-dir))
      (when filepath
        (neotree-find filepath)))))
(define-key global-map (kbd "M-e") 'my-neotree-project-dir-toggle)
```


## <span class="section-num">13</span> anki {#anki}

    ANKI_NOTE_HASH: ddecd24f60772acf8616cce396d072b4

    ANKI_NOTE_ID: 1703514191896

    ID: 6D6E9542-0E37-4C37-B6E7-9749472CE956

```emacs-lisp
(use-package anki-helper
  :vc (:fetcher github :repo Elilif/emacs-anki-helper)
  :config
  (setq anki-helper-media-directory "~/Library/Application Support/Anki2/User 1/collection.media/")
  )
```

-   Anki GUI 默认使用的 media-directory 是 "~/Library/Application Support/Anki2/User
    1/collection.media/"，需要与 anki-helper-media-directory 变量配置的一致，否则不能显示图片。
-   anki 支持启动时通过命令行参数或环境变量来指定数据目录：
    <https://docs.ankiweb.net/files.html#startup-options>
    -   数据目录下可以有多个 profile，然后使用 -p &lt;name&gt; 来指定具体的 profile name；

-   安装 anki 插件 [AnkiConnect](https://github.com/FooSoft/anki-connect)；
-   安装 anki 插件 [anki-open-org-note](https://github.com/hwiorn/anki-open-org-note)，以支持在 anki 卡片反向链接到 org 文档。

修改 anki-open-org-note 的配置： Tools -&gt; Add-ons -&gt; Open Org Note -&gt; Config, 贴入如下内容（重启anki
生效）：

-   在 org-paths 列表中添加 org-mode 文件的上层路径；
-   系统安装 ripgrep 命令，搜索命令更好；
-   修正原始的 note_match 正则表达式(最新 anki-open-org-note 版本已修复)；
-   Emacs 需要以 server 启动，而且需要关闭 socks5 代理，否则报错 ‘anki-helper--curl-sentinel: End of
    file while parsing JSON’。

<!--listend-->

```text
{
    "exec": "emacsclient -nce '(progn (select-frame-set-input-focus (selected-frame)) (find-file \"{org_file}\") (goto-char {char_pos_end}) (when (string-equal \"{note_type}\" \"ANKI_NOTE_ID\") (org-back-to-heading t)) (recenter))'",
    "note_match": "^\\s*(?:#\\+|:)(ANKI_NOTE_ID|ATTR_ID):\\s*?{note_id}\\b",
    "org-paths": [
        "~/org",
        "~/docs",
        "~/work/docs",
        "~/.emacs.d"
    ],
    "ripgrep_opts": "rg -ni",
    "shortcuts": {
        "open": "Ctrl+O"
    },
    "texts": {
        "open": "Open Note",
        "open_btn": "Open",
        "open_hint": "Open Org Note in Editor",
        "open_menu": "Org Notes"
    },
    "use_index_cache": true,
    "use_ripgrep": true
}
```

调试 anki 程序（如打印插件日志）：

```shell
zj@a:~$ cd /Applications/Anki.app/Contents/
CodeResources   Frameworks/     Info.plist      MacOS/          Resources/      _CodeSignature/
zj@a:~$ cd /Applications/Anki.app/Contents/MacOS/
zj@a:/Applications/Anki.app/Contents/MacOS$ ls
anki*  lib/  libankihelper.dylib
zj@a:/Applications/Anki.app/Contents/MacOS$ ./anki
Anki starting...
Initial setup...
Preparing to run...
Qt debug: doh set to ""  --  SystemOnly
Starting main loop...
JS warning /_anki/js/editor.js:438 No version information available for component [tex]/noerrors
JS warning /_anki/js/editor.js:438 No version information available for component [tex]/mathtools
JS warning /_anki/js/editor.js:438 No version information available for component [tex]/mhchem
regex parse error:
    ^\s*(?:#\+|:)(ANKI_NOTE_ID|ATTR_ID):\s*\"?1703514630494\"?\b
                                           ^^
error: unrecognized escape sequence
regex parse error:
    ^\s*(?:#\+|:)(ANKI_NOTE_ID|ATTR_ID):\s*\"?1703514630494\"?\b
                                           ^^
error: unrecognized escape sequence
regex parse error:
    ^\s*(?:#\+|:)(ANKI_NOTE_ID|ATTR_ID):\s*\"?1703514630494\"?\b
                                           ^^
error: unrecognized escape sequence
^Czj@a:/Applications/Anki.app/Contents/MacOS$ ./anki
```

anki-helpers 支持创建两类卡片：

1.  使用光标所在的 org-mode entry：headline 为 front 内容，下面的内容为 back 卡片内容；
2.  交互式创建 front 和 back 卡片内容；

也可以通过 entry 匹配的模式来为文档所有 entry 生成卡片，匹配 tags/property/todo 的语法于 agenda tags
view 一致：

-   例如下面的 ANKI_MATCH 只会创建两个卡片。

<!--listend-->

```text
#+ANKI_DECK: Default
#+ANKI_MATCH: TODO="TODO"|+DATE="today"
#+ANKI_NOTE_TYPE: Basic
#+ANKI_TAGS: test

* test note 1
back side
* TODO test note 2
back side
* test note 3
:PROPERTIES:
:DATE:     today
:END:
back side
* test note 4
back side
```

使用 anki-helper-set-front-region 和 anki-helper-make-two-sided-card 函数来交互式创建卡片的标题和内容：

1.  先选中内容，然后执行 anki-helper-set-front-region，设置 front 卡片内容；
2.  再选中内容，然后执行 anki-helper-make-two-sided-card，设置 back 卡片内容；

anki-helpers 默认使用 org-mode 的强调 `*xxx*` 来生成填空型卡片。

参考：

1.  [Power up Anki with Emacs, Org mode, anki-editor and more](https://yiufung.net/post/anki-org/)
2.  [Introduction - Anki Manual](https://docs.ankiweb.net/intro.html)


## <span class="section-num">14</span> project {#project}

```emacs-lisp
(use-package project
  :custom
  (project-switch-commands
   '(
     (consult-project-buffer "buffer" ?b)
     (project-dired "dired" ?d)
     (magit-project-status "magit status" ?g)
     (project-find-file "find file" ?p)
     (consult-ripgrep "rigprep" ?r)
     (vterm-toggle-cd "vterm" ?t)))
  (compilation-always-kill t)
  (project-vc-merge-submodules nil)
  :config
  ;; project-find-file 忽略的目录或文件列表。
  (add-to-list 'vc-directory-exclusion-list "vendor")
  (add-to-list 'vc-directory-exclusion-list "node_modules")
  (add-to-list 'vc-directory-exclusion-list "target"))

(defun my/project-try-local (dir)
  "Determine if DIR is a non-Git project."
  (catch 'ret
    (let ((pr-flags '(;; 顺着目录 top-down 查找第一个匹配的文件。所以中间目录不能有 .project 等文件，
		        ;; 否则判断 project root 失败。
		      ("go.mod" "Cargo.toml" "pom.xml" "package.json" ".project" )
                      ;; 以下文件容易导致 project root 判断失败, 故关闭。
                      ;; ("Makefile" "README.org" "README.md")
                      )))
      (dolist (current-level pr-flags)
        (dolist (f current-level)
          (when-let ((root (locate-dominating-file dir f)))
            (throw 'ret (cons 'local root))))))))
(setq project-find-functions '(my/project-try-local project-try-vc))

(cl-defmethod project-root ((project (head local)))
  (cdr project))

(defun my/project-discover ()
  (interactive)
  ;; 去掉 "~/go/src/k8s.io/*" 目录。
  (dolist (search-path '("~/go/src/github.com/*" "~/go/src/github.com/*/*" "~/go/src/gitlab.*/*/*"))
    (dolist (file (file-expand-wildcards search-path))
      (when (file-directory-p file)
        (message "dir %s" file)
        ;; project-remember-projects-under 列出 file 下的目录, 分别加到 project-list-file 中。
        (project-remember-projects-under file nil)
        (message "added project %s" file)))))

;; 不将 tramp 项目记录到 projects 文件中，防止 emacs-dashboard 启动时检查 project 卡住。
(defun my/project-remember-advice (fn pr &optional no-write)
  (let* ((remote? (file-remote-p (project-root pr)))
         (no-write (if remote? t no-write)))
    (funcall fn pr no-write)))
(advice-add 'project-remember-project :around 'my/project-remember-advice)
```


## <span class="section-num">15</span> terminal {#terminal}


### <span class="section-num">15.1</span> vterm {#vterm}

```bash
which cmake || brew install cmake
which glibtool || brew install libtool
which exiftran || brew install fxiftran
```

```emacs-lisp
(use-package vterm
  :hook
  (vterm-mode . (lambda ()
		  ;; 关闭一些 mode，提升显示性能。
		  (setf truncate-lines nil)
		  (setq-local show-paren-mode nil)
		  (setq-local global-hl-line-mode nil)
	          (display-line-numbers-mode -1) ;; 不显示行号。
		  ;;(font-lock-mode -1) ;; 不显示字体颜色。
		  ;;(yas-minor-mode -1)
		  ;; vterm buffer 使用 fixed pitch 的 mono 字体，否则部分终端表格之类的程序会对不齐。
		  (set (make-local-variable 'buffer-face-mode-face) 'fixed-pitch)
		  (buffer-face-mode t)))
  :config
  (setq vterm-set-bold-hightbright t)
  (setq vterm-always-compile-module t)
  (setq vterm-max-scrollback 100000)
  (setq vterm-timer-delay 0.01) ;; nil: no delay
  (add-to-list 'vterm-tramp-shells '("ssh" "/bin/bash"))
  ;; vterm buffer 名称，%s 为 shell 的 PROMPT_COMMAND 变量的输出。
  (setq vterm-buffer-name-string "*vt: %s")
  ;; 使用 M-y(consult-yank-pop) 粘贴剪贴板历史中的内容。
  (define-key vterm-mode-map [remap consult-yank-pop] #'vterm-yank-pop)
  (define-key vterm-mode-map (kbd "C-l") nil)
  ;; 防止输入法切换冲突。
  (define-key vterm-mode-map (kbd "C-\\") nil))

(use-package multi-vterm
  :after (vterm)
  :config
  (define-key vterm-mode-map  [(control return)] #'multi-vterm))

(use-package vterm-toggle
  :after (vterm)
  :custom
  ;; 由于 TRAMP 模式下关闭了 projectile，scope 不能设置为 'project。
  ;;(vterm-toggle-scope 'dedicated)
  (vterm-toggle-scope 'project)
  :config
  (global-set-key (kbd "C-`") 'vterm-toggle)
  (global-set-key (kbd "C-M-`") 'vterm-toggle-cd)
  (define-key vterm-mode-map (kbd "M-RET") #'vterm-toggle-insert-cd)
  ;; 切换到一个空闲的 vterm buffer 并插入一个 cd 命令， 或者创建一个新的 vterm buffer 。
  (define-key vterm-mode-map (kbd "s-i") 'vterm-toggle-cd-show)
  (define-key vterm-mode-map (kbd "s-n") 'vterm-toggle-forward)
  (define-key vterm-mode-map (kbd "s-p") 'vterm-toggle-backward)
  (define-key vterm-copy-mode-map (kbd "s-i") 'vterm-toggle-cd-show)
  (define-key vterm-copy-mode-map (kbd "s-n") 'vterm-toggle-forward)
  (define-key vterm-copy-mode-map (kbd "s-p") 'vterm-toggle-backward))

;; vterm-extra 提供了 vterm buffer 命令行编辑的能力，结束后按 C-c C-c 自动粘贴到对应的 vterm 中。
(use-package vterm-extra
  :vc (:fetcher github :repo Sbozzolo/vterm-extra)
  :config
  (define-key vterm-mode-map (kbd "C-c C-e") #'vterm-extra-edit-command-in-new-buffer))
```


### <span class="section-num">15.2</span> eshell {#eshell}

```emacs-lisp
(setq eshell-history-size 300)
(setq explicit-shell-file-name "/bin/bash")
(setq shell-file-name "/bin/bash")
(setq shell-command-prompt-show-cwd t)
(setq explicit-bash-args '("--noediting" "--login" "-i"))
;; 提示符只读
(setq comint-prompt-read-only t)
;; 命令补全
(setq shell-command-completion-mode t)
;; 高亮模式
(autoload 'ansi-color-for-comint-mode-on "ansi-color" nil t)
(add-hook 'shell-mode-hook 'ansi-color-for-comint-mode-on t)
(setenv "SHELL" shell-file-name)
(setenv "ESHELL" "bash")
(add-hook 'comint-output-filter-functions 'comint-strip-ctrl-m)

;; 在当前窗口右侧拆分出两个子窗口并固定，分别为一个 eshell 和当前 buffer 。
(defun my/split-windows()
  "Split windows my way."
  (interactive)
  (split-window-right 150)
  (other-window 1)
  (split-window-below)
  (eshell)
  (other-window -1)
  ;; never open any buffer in window with shell
  (set-window-dedicated-p (nth 1 (window-list)) t)
  (set-window-dedicated-p (nth 2 (window-list)) t))
(global-set-key (kbd "C-s-`") 'my/split-windows)

;; 在当前 frame 下方打开或关闭 eshell buffer。
(defun startup-eshell ()
  "Fire up an eshell buffer or open the previous one"
  (interactive)
  (if (get-buffer-window "*eshell*<42>")
      (delete-window (get-buffer-window "*eshell*<42>"))
    (progn
      (eshell 42))))
(global-set-key (kbd "s-`") 'startup-eshell)

(add-to-list 'display-buffer-alist
	     '("\\*eshell\\*<42>"
	       (display-buffer-below-selected display-buffer-at-bottom)
	       (inhibit-same-window . t)
	       (window-height . 0.33)))

;; eshell history 使用 consult-history。
(load-library "em-hist.el")
(keymap-set eshell-hist-mode-map "C-s" #'consult-history)
(keymap-set eshell-hist-mode-map "C-r" #'consult-history)
;; 重置 M-r/s 快捷键，这样 consult-line 等可用。
(define-key eshell-hist-mode-map (kbd "M-r") nil)
(define-key eshell-hist-mode-map (kbd "M-s") nil)
```


### <span class="section-num">15.3</span> tramp {#tramp}

```emacs-lisp
(use-package tramp
  :config
  ;; 使用远程主机自己的 PATH(默认是本地的 PATH)
  (setq tramp-remote-path '(tramp-default-remote-path "/bin" "/usr/bin" "/sbin" "/usr/sbin" "/usr/local/bin" "/usr/local/sbin"))
  ;;(add-to-list 'tramp-remote-path 'tramp-own-remote-path)
  ;; 使用 ~/.ssh/config 中的 ssh 持久化配置。（Emacs 默认复用连接，但不持久化连接）
  (setq tramp-use-ssh-controlmaster-options nil)
  (setq  tramp-ssh-controlmaster-options nil)
  ;; TRAMP buffers 关闭 version control, 防止卡住。
  (setq vc-ignore-dir-regexp (format "\\(%s\\)\\|\\(%s\\)" vc-ignore-dir-regexp tramp-file-name-regexp))
  ;; 关闭自动保存 ad-hoc proxy 代理配置, 防止为相同 IP 的 VM 配置了错误的 Proxy.
  (setq tramp-save-ad-hoc-proxies nil)
  ;; 调大远程文件名过期时间（默认 10s), 提高查找远程文件性能.
  (setq remote-file-name-inhibit-cache 1800)
  ;; 设置 tramp-verbose 10 打印详细信息。
  (setq tramp-verbose 1)
  ;; 增加压缩传输的文件起始大小（默认 4KB），否则容易出错： “gzip: (stdin): unexpected end of file”
  (setq tramp-inline-compress-start-size (* 1024 8))
  ;; 当文件大小超过 tramp-copy-size-limit 时，用 external methods(如 scp）来传输，从而大大提高拷贝效率。
  (setq tramp-copy-size-limit (* 1024 100))
  (setq tramp-allow-unsafe-temporary-files t)
  ;; 本地不保存 tramp 备份文件。
  (setq tramp-backup-directory-alist `((".*" .  nil)))
  ;; Backup (file~) disabled and auto-save (#file#) locally to prevent delays in editing remote files
  ;; https://stackoverflow.com/a/22077775
  (add-to-list 'backup-directory-alist (cons tramp-file-name-regexp nil))
  ;; 临时目录中保存 TRAMP auto-save 文件, 重启后清空，防止启动时 tramp 扫描文件卡住。
  (setq tramp-auto-save-directory temporary-file-directory)
  ;; 连接历史文件。
  (setq tramp-persistency-file-name (expand-file-name "tramp-connection-history" user-emacs-directory))
  ;; 避免在 shell history 中添加过多 vterm 自动执行的命令。
  (setq tramp-histfile-override nil)
  ;; 在整个 Emacs session 期间保存 SSH 密码.
  (setq password-cache-expiry nil)
  (setq tramp-default-method "ssh")
  (setq tramp-default-remote-shell "/bin/bash")
  (setq tramp-encoding-shell "/bin/bash")
  (setq tramp-default-user "root")
  (setq tramp-terminal-type "tramp")
  (customize-set-variable 'tramp-encoding-shell "/bin/bash")
  (add-to-list 'tramp-connection-properties '("/ssh:" "remote-shell" "/bin/bash"))
  (setq tramp-connection-local-default-shell-variables
        '((shell-file-name . "/bin/bash")
          (shell-command-switch . "-c")))

  ;; 自定义远程环境变量。
  (let ((process-environment tramp-remote-process-environment))
    ;; 设置远程环境变量 VTERM_TRAMP, 远程机器的 emacs_bashrc 根据这个变量设置 VTERM 参数。
    (setenv "VTERM_TRAMP" "true")
    (setq tramp-remote-process-environment process-environment)))

;; 切换 Buffer 时设置 VTERM_HOSTNAME 环境变量为多跳的最后一个主机名，并通过 vterm-environment 传递到远程 vterm shell 环境变量中，
;; 这样远程机器 ~/.bashrc 读取并执行的 emacs_bashrc 脚本正确设置 Buffer 名称和 vtem_prompt_end 函数, 从而确保目录跟踪功能正常,
;; 以及通过主机名而非 IP 来打开远程 vterm shell, 确保 SSH ProxyJump 功能正常（只能通过主机名而非 IP 访问），以及避免目标 IP 重复时
;; 连接复用错误的问题。
(defvar my/remote-host "")
(add-hook 'buffer-list-update-hook
          (lambda ()
            (when (file-remote-p default-directory)
              (setq my/remote-host (file-remote-p default-directory 'host))
              ;; 动态计算 ENV=VALUE.
              (require 'vterm)
              (setq vterm-environment `(,(concat "VTERM_HOSTNAME=" my/remote-host))))))

(use-package consult-tramp
  :vc (:fetcher github :repo Ladicle/consult-tramp)
  :custom
  ;; 默认为 scpx 模式，不支持 SSH 多跳 Jump。
  (consult-tramp-method "ssh")
  ;; 打开远程的 /root 目录，而非 ~, 避免 tramp hang。
  ;; https://lists.gnu.org/archive/html/bug-gnu-emacs/2007-07/msg00006.html
  (consult-tramp-path "/root/")
  ;; 即使 ~/.ssh/config 正确 Include 了 hosts 文件，这里还是需要配置，因为 consult-tramp 不会解析 Include 配置。
  (consult-tramp-ssh-config "~/work/proxylist/hosts_config"))
```

-   `tramp-default-method` 缺省值为 scp, 不支持多跳（但拷贝大文件时性能更高），再打开多跳远程文件时每次都需要修改
    /- 中的 -为 ssh，较麻烦，所以设置为 ssh。
-   tramp 打开远程文件时，避免使用 ~ 路径，而应该是绝对路径，[防止切换 buffer 时卡住](https://lists.gnu.org/archive/html/bug-gnu-emacs/2007-07/msg00006.html);
-   修改 net/tramp-sh.el 中的 tramp-send-commad, 将 (concat "exec env TERM='%s' INSIDE_EMACS='%s' " "ENV=%s %s
    PROMPT_COMMAND='' PS1=%s PS2='' PS3='' %s %s") 中最后的 "-i" 去掉， 然后删除同目录下的 tramp-sh.elc 文件；


## <span class="section-num">16</span> others {#others}

```bash
# 使用 GNU 系列替换 MacOS 自带的 BSD 风格核心二进制：
which tac || brew install coreutils
which trash || brew install trash
```

```emacs-lisp
;;; dired
(setq my-coreutils-path "/usr/local/opt/coreutils/libexec/gnubin")
(setenv "PATH" (concat my-coreutils-path ":" (getenv "PATH")))
(setq exec-path (cons my-coreutils-path  exec-path))
(use-package emacs
  :config
  (setq dired-dwim-target t)
  ;; @see https://emacs.stackexchange.com/questions/5649/sort-file-names-numbered-in-dired/5650#5650
  ;; 下面的参数只对安装了 coreutils (brew install coreutils) 的包有效，否则会报错。
  (setq dired-listing-switches "-laGh1v --group-directories-first"))
(use-package diredfl :config (diredfl-global-mode))

;;; diff
(use-package diff-mode
  :init
  (setq diff-default-read-only t)
  (setq diff-advance-after-apply-hunk t)
  (setq diff-update-on-the-fly t))

(use-package ediff
  :config
  (setq ediff-keep-variants nil)
  (setq ediff-split-window-function 'split-window-horizontally)
  ;; 不创建新的 frame 来显示 Control-Panel。
  (setq ediff-window-setup-function #'ediff-setup-windows-plain))

(use-package grep
  :config
  (setq grep-highlight-matches t)
  (setq grep-find-ignored-directories
	(append (list ".git" ".cache" "vendor" "node_modules" "target")
      	 grep-find-ignored-directories))
  (setq grep-find-ignored-files
	(append (list "*.blob" "*.gz" "TAGS" "projectile.cache" "GPATH" "GRTAGS" "GTAGS" "TAGS" ".project" )
      	 grep-find-ignored-files)))

(global-set-key "\C-cn" 'find-dired)
(global-set-key "\C-cN" 'grep-find)

(setq isearch-allow-scroll 'unlimited)
;; 显示当前和总的数量。
(setq isearch-lazy-count t)
(setq isearch-lazy-highlight t)

;;在线搜索, 可以先选中 region 再执行搜索。
(use-package engine-mode
  :config
  (engine/set-keymap-prefix (kbd "C-c s"))
  (engine-mode t)
  ;;(setq engine/browser-function 'eww-browse-url)
  (defengine github "https://github.com/search?ref=simplesearch&q=%s" :keybinding "h")
  (defengine google "http://www.google.com/search?ie=utf-8&oe=utf-8&q=%s" :keybinding "g"))

;;; Google 翻译
(use-package google-translate
  :config
  (setq max-mini-window-height 0.2)
  ;; C-n/p 切换翻译类型。
  (setq google-translate-translation-directions-alist
        '(("en" . "zh-CN") ("zh-CN" . "en")))
  (global-set-key (kbd "C-c d t") #'google-translate-smooth-translate))

;;; xwidget
;;Emacs 29 的 xwidget-webkit 对 Mac 支持不好(
;;[[https://github.com/d12frosted/homebrew-emacs-plus/issues/519][Better support for
;;xwidget-webkit]]), 部分功能只支持GTK/X11 版本, 如: increase-search/webkit-history:
(setq url-user-agent "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.71 Safari/537.36")
(setq xwidget-webkit-buffer-name-format "*webkit* [%T] - %U")
(setq xwidget-webkit-enable-plugins t)
(setq browse-url-firefox-program "/Applications/Firefox.app/Contents/MacOS/firefox")
(setq browse-url-browser-function 'xwidget-webkit-browse-url) ;; browse-url-firefox, browse-url-default-macosx-browser
(setq xwidget-webkit-cookie-file "~/.emacs.d/cookie.txt")

(add-hook 'xwidget-webkit-mode-hook
          (lambda ()
            (setq kill-buffer-query-functions nil)
            (setq header-line-format nil)
            (display-line-numbers-mode 0)
            (local-set-key "q" (lambda () (interactive) (kill-this-buffer)))
            (local-set-key (kbd "C-t") (lambda () (interactive) (xwidget-webkit-browse-url "https://google.com" t)))))

(defun my/browser-open-at-point (url)
  (interactive
   (list (let ((url (thing-at-point 'url)))
           (if (equal major-mode 'xwidget-webkit-mode)
               (read-string "url: " (xwidget-webkit-uri (xwidget-webkit-current-session)))
             (read-string "url: " url)))))
  (xwidget-webkit-browse-url url t))

(defun my/browser-google (query)
  (interactive "ssearch: ")
  (xwidget-webkit-browse-url
   (concat "https://google.com/search?q=" (string-replace " " "%20" query)) t))

(define-prefix-command 'my-browser-prefix)
(global-set-key (kbd "C-c o") 'my-browser-prefix)
(define-key my-browser-prefix (kbd "o") 'my/browser-open-at-point)
(define-key my-browser-prefix (kbd "g") 'my/browser-google)

;; 保存 Buffer 时自动更新 #+LASTMOD: 时间戳。
(setq time-stamp-start "#\\+\\(LASTMOD\\|lastmod\\):[ \t]*")
(setq time-stamp-end "$")
(setq time-stamp-format "%Y-%m-%dT%02H:%02m:%02S%5z")
;; #+LASTMOD: 必须位于文件开头的 line-limit 行内, 否则自动更新不生效。
(setq time-stamp-line-limit 30)
(add-hook 'before-save-hook 'time-stamp t)

(use-package emacs
  :init
  ;; 粘贴于光标处, 而不是鼠标指针处。
  (setq mouse-yank-at-point t)
  (setq initial-major-mode 'fundamental-mode)
  ;; 按中文折行。
  (setq word-wrap-by-category t)
  ;; 退出自动杀掉进程。
  (setq confirm-kill-processes nil)
  (setq use-short-answers t)
  (setq confirm-kill-emacs #'y-or-n-p)
  (setq ring-bell-function 'ignore)
  ;; 不显示行号, 否则鼠标会飘。
  (add-hook 'artist-mode-hook (lambda () (display-line-numbers-mode -1)))
  ;; bookmark 发生变化时自动保存（默认是 Emacs 正常退出时保存）。
  (setq bookmark-save-flag 1)
  ;; 不创建 lock 文件。
  (setq create-lockfiles nil)
  ;; 启动 Server 。
  (unless (and (fboundp 'server-running-p)
               (server-running-p))
    (server-start)))

(use-package ibuffer
  :config
  (setq ibuffer-expert t)
  (setq ibuffer-use-other-window nil)
  (setq ibuffer-movement-cycle nil)
  (setq ibuffer-default-sorting-mode 'recency)
  (setq ibuffer-use-header-line t)
  (add-hook 'ibuffer-mode-hook #'hl-line-mode)
  (global-set-key (kbd "C-x C-b") #'ibuffer))

(use-package recentf
  :config
  (setq recentf-save-file "~/.emacs.d/recentf")
  ;; 不自动清理 recentf 记录。
  (setq recentf-auto-cleanup 'never)
  ;; emacs 退出时清理 recentf 记录。
  (add-hook 'kill-emacs-hook #'recentf-cleanup)
  ;; 每 5min 以及 emacs 退出时保存 recentf-list。
  ;;(run-at-time nil (* 5 60) 'recentf-save-list)
  ;;(add-hook 'kill-emacs-hook #'recentf-save-list)
  (setq recentf-max-menu-items 100)
  (setq recentf-max-saved-items 200) ;; default 20
  ;; recentf-exclude 的参数是正则表达式列表，不支持 ~ 引用家目录。
  ;; emacs-dashboard 不显示这里排除的文件。
  (setq recentf-exclude `(,(recentf-expand-file-name "~\\(straight\\|ln-cache\\|etc\\|var\\|.cache\\|backup\\|elfeed\\)/.*")
                          ,(recentf-expand-file-name "~\\(recentf\\|bookmarks\\|archived.org\\)")
                          ,tramp-file-name-regexp ;; 不在 recentf 中记录 tramp 文件，防止 tramp 扫描时卡住。
                          "^/tmp" "\\.bak\\'" "\\.gpg\\'" "\\.gz\\'" "\\.tgz\\'" "\\.xz\\'" "\\.zip\\'" "^/ssh:" "\\.png\\'"
                          "\\.jpg\\'" "/\\.git/" "\\.gitignore\\'" "\\.log\\'" "COMMIT_EDITMSG" "\\.pyi\\'" "\\.pyc\\'"
                          "/private/var/.*" "^/usr/local/Cellar/.*" ".*/vendor/.*"
                          ,(concat package-user-dir "/.*-autoloads\\.egl\\'")))
  (recentf-mode +1))

(defvar backup-dir (expand-file-name "~/.emacs.d/backup/"))
(if (not (file-exists-p backup-dir))
    (make-directory backup-dir t))
;; 文件第一次保存时备份。
(setq make-backup-files t)
(setq backup-by-copying t)
;; 不备份 tramp 文件，其它文件都保存到 backup-dir, https://stackoverflow.com/a/22077775
(setq backup-directory-alist `((,tramp-file-name-regexp . nil) (".*" . ,backup-dir)))
;; 备份文件时使用版本号。
(setq version-control t)
;; 删除过多的版本。
(setq delete-old-versions t)
(setq kept-new-versions 6)
(setq kept-old-versions 2)

(defvar autosave-dir (expand-file-name "~/.emacs.d/autosave/"))
(if (not (file-exists-p autosave-dir))
    (make-directory autosave-dir t))
;; auto-save 访问的文件。
(setq auto-save-default t)
(setq auto-save-list-file-prefix autosave-dir)
(setq auto-save-file-name-transforms `((".*" ,autosave-dir t)))

(global-auto-revert-mode 1)
(setq revert-without-query (list "\\.png$" "\\.svg$")
      auto-revert-verbose nil)

(setq global-mark-ring-max 100)
(setq mark-ring-max 100 )
(setq kill-ring-max 100)

;; minibuffer 历史记录。
(use-package savehist
  :hook (after-init . savehist-mode)
  :config
  (setq history-length 600)
  (setq savehist-save-minibuffer-history t)
  (setq savehist-autosave-interval 200)
  (add-to-list 'savehist-additional-variables 'mark-ring)
  (add-to-list 'savehist-additional-variables 'global-mark-ring)
  (add-to-list 'savehist-additional-variables 'extended-command-history))

;; 使用系统剪贴板，实现与其它程序相互粘贴。
(setq x-select-enable-clipboard t)
(setq select-enable-clipboard t)
(setq x-select-enable-primary t)
(setq select-enable-primary t)

;; UTF8 字符。
(prefer-coding-system 'utf-8)
(setq locale-coding-system 'utf-8
      default-buffer-file-coding-system 'utf-8)
(set-buffer-file-coding-system 'utf-8)
(set-language-environment "UTF-8")
(set-default buffer-file-coding-system 'utf8)
(set-default-coding-systems 'utf-8)
(setenv "LC_ALL" "zh_CN.UTF-8")

;; 删除文件时, 将文件移动到回收站。
(use-package osx-trash
  :config
  (when (eq system-type 'darwin)
    (osx-trash-setup))
  (setq-default delete-by-moving-to-trash t))

;; 在 Finder 中打开当前文件。
(use-package reveal-in-osx-finder
  :commands (reveal-in-osx-finder))

;; 在帮助文档底部显示 lisp demo.
(use-package elisp-demos
  :config
  (advice-add 'describe-function-1 :after #'elisp-demos-advice-describe-function-1)
  (advice-add 'helpful-update :after #'elisp-demos-advice-helpful-update))

;; 相比 Emacs 内置 Help, 提供更多上下文信息。
(use-package helpful
  :config
  (global-set-key (kbd "C-h f") #'helpful-callable)
  (global-set-key (kbd "C-h v") #'helpful-variable)
  (global-set-key (kbd "C-h k") #'helpful-key)
  (global-set-key (kbd "C-c C-d") #'helpful-at-point)
  (global-set-key (kbd "C-h F") #'helpful-function)
  (global-set-key (kbd "C-h C") #'helpful-command))

;; 在另一个 panel buffer 中展示按键。
;;(use-package command-log-mode :commands command-log-mode)
(use-package hydra :commands defhydra)

;; 以下自定义函数参考自：https://github.com/jiacai2050/dotfiles/blob/master/.config/emacs/i-edit.el
(defun my/json-format ()
  (interactive)
  (save-excursion
    (if mark-active
        (json-pretty-print (mark) (point))
      (json-pretty-print-buffer))))

(defun my/delete-file-and-buffer (buffername)
  "Delete the file visited by the buffer named BUFFERNAME."
  (interactive "bDelete file")
  (let* ((buffer (get-buffer buffername))
         (filename (buffer-file-name buffer)))
    (when filename
      (delete-file filename)
      (message "Deleted file %s" filename)
      (kill-buffer))))

(defun my/diff-buffer-with-file ()
  "Compare the current modified buffer with the saved version."
  (interactive)
  (let ((diff-switches "-u")) ;; unified diff
    (diff-buffer-with-file (current-buffer))
    (other-window 1)))

(defun my/copy-current-filename-to-clipboard ()
  "Copy `buffer-file-name' to system clipboard."
  (interactive)
  (let ((filename (if-let (f buffer-file-name)
                      f
                    default-directory)))
    (if filename
        (progn
          (message (format "Copying %s to clipboard..." filename))
          (kill-new filename))
      (message "Not a file..."))))

;; https://gitlab.com/skybert/my-little-friends/-/blob/2022-emacs-from-scratch/emacs/.emacs
;; Rename current buffer, as well as doing the related version control
;; commands to rename the file.
(defun my/rename-this-buffer-and-file ()
  "Renames current buffer and file it is visiting."
  (interactive)
  (let ((filename (buffer-file-name)))
    (if (not (and filename (file-exists-p filename)))
        (message "Buffer is not visiting a file!")
      (let ((new-name (read-file-name "New name: " filename)))
        (cond
         ((vc-backend filename) (vc-rename-file filename new-name))
         (t
          (rename-file filename new-name t)
          (rename-buffer new-name)
          (set-visited-file-name new-name)
          (set-buffer-modified-p nil)
          (message
           "File '%s' successfully renamed to '%s'"
           filename
           (file-name-nondirectory new-name))))))))
(global-set-key (kbd "C-x C-r") 'my/rename-this-buffer-and-file)
```

-   osx-trash 不支持 TRAMP 删除远程文件，解决办法：用 %m 标记文件，然后按 ! 执行 rm 命令。
-   参考： [Mastering Key Bindings in Emacs](https://www.masteringemacs.org/article/mastering-key-bindings-emacs)


## <span class="section-num">17</span> refs {#refs}

    ANKI_NOTE_HASH: 64901ec4f38ae35f38b42523c75cb9ea

    ANKI_NOTE_ID: 1703514360198

本配置参考了以下仓库代码：

1.  [seagle0128/.emacs.d](https://github.com/seagle0128/.emacs.d)
2.  [protesilaos/dotfiles](https://gitlab.com/protesilaos/dotfiles)
3.  [bbatsov/prelude](https://github.com/bbatsov/prelude)
4.  [MatthewZMD/.emacs.d](https://github.com/MatthewZMD/.emacs.d)
5.  [condy0919/.emacs.d](https://github.com/condy0919/.emacs.d)
6.  [manateelazycat/lazycat-emacs](https://github.com/manateelazycat/lazycat-emacs)
7.  [jiacai2050/dotfiles](https://github.com/jiacai2050/dotfiles)
8.  [natecox/dotfiles](https://github.com/natecox/dotfiles/)
9.  [daviwil](https://config.daviwil.com/emacs)
10. [skybert/my-little-friends](https://gitlab.com/skybert/my-little-friends/-/blob/master/emacs/.emacs)
11. [casouri/lunarymacs](https://github.com/casouri/lunarymacs/commits/master)
