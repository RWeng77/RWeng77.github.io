# How to Customize Your Blog

Welcome to the customization guide for your Jekyll Chirpy blog! This document will serve as your cheat sheet for knowing exactly which files to edit when you want to change parts of your website.

## 1. Changing Colors (Light & Dark Mode)
Edit `assets/css/jekyll-theme-chirpy.scss`. It shadows the theme's entry stylesheet and appends
colour overrides after the theme is loaded.

* **Light Mode:** put variables inside the `light-colors` mixin.
* **Dark Mode:** put variables inside the `dark-colors` mixin.

Uncomment or add any CSS variable (`--main-bg`, `--text-color`, `--sidebar-bg`, `--link-color`, ...).
The full list of variable names lives in the gem: run `bundle show jekyll-theme-chirpy`, then open
`_sass/themes/_light.scss` and `_sass/themes/_dark.scss` there.

Note: creating `_sass/themes/_light.scss` in this repo does **not** work — the theme loads its
colour files with a gem-relative `@use '../themes/light'`, so a copy here is never picked up.

### Hidden "Miku" skin
Type `miku` anywhere on a page (not inside the search box) to toggle it; `#miku` / `#nomiku` on any
URL also switches it on/off. The choice is remembered in `localStorage`. Colours live in the
`:root[data-miku]` block at the bottom of `assets/css/jekyll-theme-chirpy.scss`; the trigger is
`_includes/metadata-hook.html`; the pixel font is `assets/fonts/fusion-pixel-12px-zh_hant.woff2`
(Fusion Pixel, OFL — licence in the same folder).

## 2. Basic Configuration (Domain, Avatar, Title)
Almost all global settings for your site live in one single file: `_config.yml`.
Open `_config.yml` and look for:
* `url:` The domain of your website (Must include `https://`).
* `title:` The main text that shows up in the browser tab and sidebar.
* `tagline:` The subtitle shown underneath your name in the sidebar.
* `avatar:` The URL of your profile picture.
* `timezone:` Your timezone (make sure this is correct for post timestamps).

## 3. Social Media Links
Also in `_config.yml`, find the `social:` section. 
Here you can type your GitHub, Twitter, Email, and easily link them up so the icons appear correctly on your sidebar.

## 4. Writing New Articles (Posts)
To create a new blog post:
1. Go into the `_posts/` folder.
2. Create a new markdown file named in this exact format: `YYYY-MM-DD-title-of-post.md` (e.g., `2024-03-31-my-first-post.md`).
3. Make sure you include the "front matter" at the very top of your post! You can copy the layout from existing posts.

## 5. Editing the Built-in Pages (About, etc)
The sections on your sidebar (like Home, Categories, Tags, Archives, and About) are located in the `_tabs/` folder.
* Want to change the "About" page text? Open `_tabs/about.md` and just start typing below the dashes `---`.

## 6. Advanced Customization & Overriding (The "Golden Rule")
Because your theme is installed as a Ruby gem, many of its core files are hidden. However, you can use the **Overriding Rule**:

> **Rule:** If you create a file in your project folder with the *exact same name and folder path* as a file inside the theme gem, your file will override the theme's default file!

* **Customizing the 404 Page:** I just created `assets/404.html` for you! You can now open this file and edit your "Page Not Found" screen to say whatever you want. Since it mimics the `assets/` path of the theme, your site will use it.
* **Customizing the Sidebar:** Create `_includes/sidebar.html` and copy/paste your HTML code.
* **Customizing the Head (`<head>`):** Create `_includes/head.html`.
* **Customizing the Footer:** Create `_includes/footer.html`.

## How to Apply Changes
* **If running locally:** When you change `_config.yml`, you must **restart** your Jekyll server to see the changes. If you only change SCSS or HTML text, it will update automatically!
* **If publishing to GitHub:** Just stage, commit, and push your code. Wait ~2 minutes for GitHub Actions to complete, and your live site will reflect the changes.
