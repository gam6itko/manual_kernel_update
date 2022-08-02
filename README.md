# Описание работы

## tl;dr;

Cсылка на box в vagrant cloud: <https://app.vagrantup.com/gam6itko/boxes/centos-7-5>

Ниже описан мной проделаннй путь для выполнения задания.


## подготовка

Host machine:
```shell
uname -srv
# Linux 5.15.0-43-generic #46-Ubuntu SMP Tue Jul 12 10:30:17 UTC 2022
```

- Поставил vagrant `apt install vagrant` (Vagrant 2.2.19)
- Форкнул репозиторий и запустил `vagrant up` 

Возникла проблема 
```
Error while connecting to Libvirt: Error making a connection to libvirt URI qemu:///system:
Call to virConnectOpen failed: Failed to connect socket to '/var/run/libvirt/libvirt-sock': No such file or directory
```

Нашёл статью <https://opensource.com/article/21/10/vagrant-libvirt>. Оказалось что нужно установить daemon и плагин

```shell
sudo apt install ruby-libvirt qemu libvirt-daemon-system libvirt-clients libvirt-dev
sudo systemctl start libvirtd
vagrant plugin install vagrant-libvirt
```

Не помогло.
Оказалось, что я просто не установил `virtualbox` 🤦

Установил `virtualbox`, сделал `vagrant up`. Не смог скачать образ `centos/7`. 
Пришлось включить VPN. Образ начал скачиваться. 

...

...

...

Скачивание образа через VPN заняло 2 часа 😢

## обновление ядра

Залогинился в виртуальную машину, скачал ядро, установил ядро.

Пришло время обновить grub. Но что-то мне незнакома магия с `grub2-mkconfig`, пошел ознакамливаться с документацией.
<https://linuxhint.com/grub2_mkconfig_tutorial/>

Команды с grub сделал, перезагрузил.

Ядро обновилось
```shell
vagrant ssh
uname -r
# 5.19.0-1.el7.elrepo.x86_64
```

## packer
всё прочитанное в методичке понятно, делаем образ

```shell
cd ./packer
packer build centos.json
```

Получил вывод ниже. Похоже на ошибку
```
Error: Failed to prepare build: "centos-7.7"

1 error occurred:
        * Deprecated configuration key: 'iso_checksum_type'. Please call `packer fix`
against your template to update your template to be compatible with the current
version of Packer. Visit https://www.packer.io/docs/commands/fix/ for more
detail.


==> Wait completed after 2 microseconds

==> Builds finished but no artifacts were created.

```

Из вывода следует что нужно починить конфиг.

```shell
packer fix centos.json > centos-fixed.json
packer build centos-fixed.json
```

После запуска сбоки появились зеленые строчки похожие на вывод vagrant'a.

Потом открылось GUI virtualbox с запущенной виртуальной машиной. 

Похоже, скрипты из папки `./packer/scripts` начали выполняться.

...

Дождался финала
```
==> Wait completed after 10 minutes 32 seconds

==> Builds finished. The artifacts of successful builds are:
--> centos-7.7: 'virtualbox' provider box: centos-7.7.1908-kernel-5-x86_64-Minimal.box
```

Появился файл `./packer/centos-7.7.1908-kernel-5-x86_64-Minimal.box`

Как я понял, с помощью packer мы сделали то же самое что я делал руками в виртуальной машине.


## vagrant init (тестирование)

создал папку `test` в неё запустил:
```
vagrant init centos-7-5
vagrant up
```

произошла ошибка
```
Vagrant was unable to mount VirtualBox shared folders. This is usually
because the filesystem "vboxsf" is not available. This filesystem is
made available via the VirtualBox Guest Additions and kernel module.
Please verify that these guest additions are properly installed in the
guest. This is not a bug in Vagrant and is usually caused by a faulty
Vagrant box. For context, the command attempted was:

mount -t vboxsf -o uid=1000,gid=1000,_netdev vagrant /vagrant

The error output from the command was:

mount: unknown filesystem type 'vboxsf'
```

Попробуем починить, убрав synced_folder `config.vm.synced_folder ".", "/vagrant", disabled: true`

```shell
vagrant up
# проблем нет
vagrant ssh

uname -r
# 5.19.0-1.el7.elrepo.x86_64
```

Ядро в виртуальной машине оказалось даже новее чем то что в методичке (5.3.1-1.el7.elrepo.x86_64).

Выходим и удаляем box

```shell
vagrant box remove -f centos-7-5
# Removing box 'centos-7-5' (v0) with provider 'virtualbox'...
```

## vagrant cloud

Производим аутентификацию в vagrant cloud

```shell
vagrant cloud auth login -u gam6itko
```

ввёл пароль, появились какие-то ошибки
```
Vagrant Cloud request failed (VagrantCloud::Error::ClientError::RequestError)
```

Попробуем через VPN. Аутентификация прошла успешно.

Отправляем образ
```shell
vagrant cloud publish --release gam6itko/centos-7-5 1.0 virtualbox ./packer/centos-7.7.1908-kernel-5-x86_64-Minimal.box
```
Пишут что через VPN это займет около 3 часов 👀

После долгого ожидания, образ залился.
```
Complete! Published gam6itko/centos-7-5
Box:              gam6itko/centos-7-5
Description:      
Private:          yes
Created:          2022-08-03T11:16:22.295+03:00
Updated:          2022-08-03T11:16:26.873+03:00
Current Version:  N/A
Versions:         1.0
Downloads:        0
```

## ссылка на box в vagrant cloud

<https://app.vagrantup.com/gam6itko/boxes/centos-7-5>
