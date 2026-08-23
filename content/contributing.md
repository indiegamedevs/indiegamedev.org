---
title: Contribution Guide
summary: How to contribute games, tools, articles, tutorials, videos, news, studios, and developer profiles to indiegamedev.org.
type: blog
---

indiegamedev.org is built from plain Markdown files. Most contributions are new pages under `content/`, small updates to existing entries, or image assets added under `static/images/`.

The goal is to make each entry useful to indie game developers: clear title, short description, relevant links, accurate studio or developer attribution, and an image when one helps identify the project.

This site is meant to grow as a community reference, not a launch calendar or promotional directory. Studios should add games, tools, and resources that other developers can learn from, study, use, or build on. Open source projects are a strong fit because they can act as practical references for other indie developers.

## What to Add

Good contributions include:

- Open source games that can be used as references by other developers.
- Tools, engines, plugins, libraries, and other projects that help people make games.
- Articles, tutorials, videos, and news that are useful to indie game developers.
- Studio and developer profiles connected to listed projects.
- Corrections to dates, descriptions, images, tags, links, or attribution.

Avoid adding low-context listings. If a page only has a title and a link, it is usually worth adding a short explanation of why the project belongs here.

Upcoming commercial releases, teaser pages, and purely promotional listings are usually not a good fit. A game planned for release months from now may be important to the studio, but it does not necessarily help the community learn, compare approaches, or use the project as a reference.

Work-in-progress or placeholder entries can still be useful when they provide context for related development work. For example, a game page may explain what a studio is building so the surrounding tools, libraries, articles, or tutorials make sense. In that case, the entry should be honest about its status and focus on the development context rather than marketing copy.

When a game from a community contributor is released, it can also be submitted as a news entry. Relevant community releases may be featured on the homepage when they are useful to highlight for the broader indie game development community.

## Content Locations

Studio and developer profiles live here:

- `content/studios/`
- `content/devs/`

Project and resource entries should live inside a studio folder under the matching section:

- `content/news/`
- `content/articles/`
- `content/tutorials/`
- `content/videos/`
- `content/games/`
- `content/tools/`

For example, a studio named `example-studio` would add game entries under `content/games/example-studio/`, tool entries under `content/tools/example-studio/`, and article entries under `content/articles/example-studio/`.

Images should use the same studio folder structure under `static/images/`. For example, game images for `example-studio` should live under `static/images/games/example-studio/`, and tool images should live under `static/images/tools/example-studio/`.

Reference those images from front matter with the public `/images/` path:

```yaml
image: /images/games/example-studio/example-game.jpg
```

## Adding an Entry

1. Add or update the studio profile in `content/studios/`.
2. Add or update any related developer profile in `content/devs/`.
3. Pick the section that best fits the contribution.
4. Create a folder for the studio inside that section if one does not exist yet.
5. Create the Markdown entry inside that studio folder.
6. Add front matter with a title, summary, date when relevant, tags, studio attribution, and links.
7. Add a short description in the page body.
8. Add an image if the project has a useful logo, screenshot, cover image, or icon.

For example:

- `content/studios/example-studio.md`
- `content/devs/example-developer.md`
- `content/games/example-studio/example-game.md`
- `content/tools/example-studio/example-tool.md`
- `content/articles/example-studio/example-article.md`
- `static/images/games/example-studio/example-game.jpg`
- `static/images/tools/example-studio/example-tool.jpg`

A game entry can look like this:

```yaml
---
title: Example Game
date: 2026-08-21
summary: A short description of the game.
game_studios:
  - example-studio
game_tags:
  - puzzle
  - open-source
image: /images/games/example-studio/example-game.jpg
external_url: https://example.com
---
```

Use the matching taxonomy for the section:

- Games use `game_studios` and `game_tags`.
- Tools use `tool_studios` and `tool_tags`.
- Articles use `article_studios` and `articles_tags`.
- News uses `news_studios` and `news_tags`.
- Tutorials use `tutorial_studios` and `tutorial_tags`.
- Videos use `video_studios` and `video_tags`.

## Studios and Developers

If a project is connected to a studio or developer that is not listed yet, add a profile for them first. Studio pages belong in `content/studios/`, and developer pages belong in `content/devs/`.

After the studio exists, create matching studio folders under the sections where they are contributing content. A studio does not need every section folder. Only add the folders that contain real entries.

Keep profiles factual and concise. Mention what they create or contribute to, then link to the relevant website, source repository, or project page.

## How to Contribute with GitHub

If you are new to Git, the simplest path is to use GitHub's web editor:

1. Go to the indiegamedev.org repository on GitHub.
2. Click **Fork** to create your own copy of the repository.
3. Find the file you want to edit, or create a new file in the right `content/` folder.
4. Use the pencil icon to edit the file in your fork.
5. Commit the change in your fork with a short message, such as `Add example studio profile`.
6. Open a pull request back to the main indiegamedev.org repository.

For larger changes, images, or several related entries, it is usually easier to work locally:

```bash
git clone https://github.com/YOUR_USERNAME/indiegamedev.org.git
cd indiegamedev.org
git checkout -b feature/add-example-studio
```

Make your changes, then check what changed:

```bash
git status
git diff
```

Stage and commit the files:

```bash
git add content/studios/example-studio.md content/games/example-studio/example-game.md static/images/games/example-studio/example-game.jpg
git commit -m "Add example studio and game"
```

Push your branch to GitHub:

```bash
git push origin feature/add-example-studio
```

Then open a pull request from your branch on GitHub. The pull request should briefly explain what you added and why it is useful for indie game developers.

## Pull Request Checklist

Before opening a pull request:

- Confirm links work.
- Check names, dates, and attribution.
- Use images that are allowed to be redistributed or are already part of the linked project.
- Keep image files reasonably sized.
- Run the site locally with Hugo if it is available.

Then open a pull request on GitHub:

<p><a class="igd-post-button igd-post-button--primary" href="https://github.com/indiegamedevs/indiegamedev.org" target="_blank" rel="noopener noreferrer">Contribute on GitHub</a></p>

