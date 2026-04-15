# Overview

## .vimrc

```shell
set ai
set et
set hls
set ic
set is
set sw=2
set ts=2
```

## kubectl

```shell
# add in ~/.bashrc
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
source ~/.bashrc
```
