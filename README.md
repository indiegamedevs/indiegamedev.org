# indiegamedev.org

Open-source, Markdown-first static site for indiegamedev.org: news, articles, tutorials, and community resources for indie game devs. PRs welcome.

### GitHub Pages

[indiegamedev.org](https://indiegamedev.org)

## Local Development

Pre-requisites: [Hugo](https://gohugo.io/getting-started/installing/), [Go](https://golang.org/doc/install) and [Git](https://git-scm.com)

```shell
# Clone the repo
git clone https://github.com/indiegamedevs/indiegamedev.org.git

# Change directory
cd indiegamedev.org

# Start the server
hugo mod tidy
hugo server --logLevel debug --disableFastRender -p 8080
```

### Update theme

```shell
hugo mod get -u
hugo mod tidy
```
