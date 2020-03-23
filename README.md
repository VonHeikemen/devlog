# Devlog

Source code for the blog section of [vonheikemen.github.io](https://vonheikemen.github.io/devlog/).

## Getting started

To create content from your local environment download the [zola](https://www.getzola.org/) static site engine.

### Installation

Once you've downloaded zola clone/download this repository.

```
 git clone https://github.com/VonHeikemen/devlog 
 cd devlog
```

## Usage

### Create content

Navigate to the `content` folder and create a markdown file then open it with your favorite text editor.

```
cd devlog/content
touch some-article.md
nvim some-article.md
```

Currently the theme supports two languages english and spanish. The language is determined by a language code, you can find them in the `config.toml` file. By default the articles created without a "language code" in it's name are in english. To create a translated article you have to create another markdown file with the same name as the original but with the language code as a suffix before the `.md` extention. For example.

```
cd devlog/content
touch some-article.es.md
nvim some-article.es.md
```

### Preview content

To build a preview in your local environment you can use the `zola serve` command.

```
cd devlog
zola serve --port 3000
```

### Build the site

To build the site use the `zola build` command.

```
cd devlog
zola build --output-dir /path-to/public-folder
```

## More information

If you want to modify the theme make sure to visit the [zola documentation](https://www.getzola.org/documentation/) and the [tera site](https://tera.netlify.com/docs/) (tera is the template engine).

