# Установка Ruby из исходников (rbenv)

#### Устанавливаем *ruby*

```bash
cd
git clone git://github.com/sstephenson/rbenv.git .rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec $SHELL
```

```bash
git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
exec $SHELL
```

```bash
git clone https://github.com/sstephenson/rbenv-gem-rehash.git ~/.rbenv/plugins/rbenv-gem-rehash
```

Переходим по ссылке https://www.ruby-lang.org/ru/downloads/ и смотрим текущую (свежую) версию Ruby. Далее устанавливаем ее (в нашем случае 2.2.3)

```bash
rbenv install 2.2.3
rbenv global 2.2.3
ruby -v
```

#### Устанавливаем *bundler*

```bash
echo "gem: --no-ri --no-rdoc" > ~/.gemrc
gem install bundler
```
#### Обновляем *ruby*

```bash
cd
cd .rbenv/plugins/ruby-build/
git pull
cd
rbenv install -l
rbenv install X.Y.Z
rbenv global X.Y.Z
ruby -v
```