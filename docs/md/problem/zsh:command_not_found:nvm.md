### zsh: command not found:nvm解决办法

#### 修改bash_profile
```shell
vim ~/.bash_profile
```
```shell
export NVM_DIR="${NVM_HOME}"#${NVM_HOME}为自己的路径
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```
```shell
保存并且退出以后
source ~/.bash_profile
```

#### 在zsh中依旧报 zsh: command not found错误
```shell
vim ~/.zshrc
```
新增行
```shell
source ~/.bash_profile
```
```shell
source ~/.zshrc
```

#### 如果如果不行

在.zshrc 最底部加入这些试试：
```shell
PATH=/bin:/usr/bin:/usr/local/bin:${PATH}
export PATH
```