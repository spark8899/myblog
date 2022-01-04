# myblog
person blog

# install hugo
```
brew install hugo
hugo new site myblog
cd myblog
```

# add new md
```
hugo new xx.md
vim content/xx.md

hugo new post/first.md
```

# run hugo
```
hugo server --theme=hyde --buildDrafts
```

# create html
```
hugo --buildDrafts --theme=hyde --baseUrl="https://sparkknow.com/"
```
