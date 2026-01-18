+++
date = '2025-01-05T21:45:01+07:00'
draft = false
title = 'Всякие git хаки'
soure = 'https://stackoverflow.com/questions/4220416/can-i-specify-multiple-users-for-myself-in-gitconfig'
tags = ['git']
+++

## Разделение настроек

У гита есть 2 вида настроек:

- локальные - определяются в `.git/config`
- глобальные - определяются либо в `.config/git/config`, либо в `.gitconfig`

Если нужно сделать глобальные настройки для определнных репозиториев, можно воспользоваться диррективой `includeIf`.
В глобальные настройки гита добавляем следующее:

```
[includeIf "gitdir:~/Projects/work/"]
    path = ~/Projects/work/.gitconfig
```

Теперь если мы находимся в дирректории work, будут подключены настройки спецефичные для данных проектов

## удалить файл из git repo

```bash
git filter-repo -f --index-filter 'git rm --cached --ignore-unmatch unwanted-file.txt'
```

после этого надо пушить с форсом:

``` bash
git push --force -u origin main
```

## Перезапись автора коммитов

Перезапись с определенного коммита

``` bash
git rebase -r --root --exec "git commit --amend --no-edit --reset-author"
```

Перезапись всех коммитов

```bash
git rebase -r --root --exec "git commit --amend --no-edit --reset-author
```

> ❗ Если во время перезаписи будут происходить какие-то махинации с gitconfig, нужно автора прописать в локальные настройки

## Подсчет колличества строк в репозитории

```bash
git ls-files | xargs cloc
```

## Git home config

```bash
alias home='git --work-tree=$HOME --git-dir=$HOME/.cfg'
```
