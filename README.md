# Сбор и анализ логов 

## Домашнее задание по теме _Сбор и анализ логов_  

> в вагранте поднимаем 2 машины web и log  
> на web поднимаем nginx  
> на log настраиваем центральный лог сервер на любой системе на выбор  
> * journald;  
> * rsyslog;  
> * elk.    

> настраиваем аудит, следящий за изменением конфигов нжинкса  
> Все критичные логи с web должны собираться и локально и удаленно.  
> Все логи с nginx должны уходить на удаленный сервер (локально только критичные).  
> Логи аудита должны также уходить на удаленную систему.  
> Формат сдачи ДЗ - vagrant + ansible  

### Решение ДЗ.
1. При помощи _Vagrant+Аnsible_ разворачиваем виртуальные машины _web_ и _log_. Всё выполняется автоматически согласно методички домашнего задания.
На хосте через браузер заходим на страницу _192.168.50.10_ и видим ошибку, поскольку папка _/usr/share/nginx/html_ удалена. Для проверки логов на машине 
_log_ выполняем команды
> cat /var/log/rsyslog/web/nginx_access.log  
> cat /var/log/rsyslog/web/nginx_error.log 

и видим, что логи пишутся корректно:

![](https://github.com/Vitaliy7/logs/blob/main/screenshots/log.png?raw=true)

2. Настройка аудита, контролирующего изменения конфигурации _nginx_  
  За аудит отвечает уже установленная утилита _auditd_. Настроим аудит изменения конфигурации _nginx_:
Добавим правило, которое будет отслеживать изменения в конфигруации _nginx_. Для этого в конец
файла 
> /etc/audit/rules.d/audit.rules 

добавим следующие строки:

> -w /etc/nginx/nginx.conf -p wa -k nginx_conf  
> -w /etc/nginx/default.d/ -p wa -k nginx_conf  

![](https://github.com/Vitaliy7/logs/blob/main/screenshots/audit.png?raw=true)

Перезапускаем службу  
> auditd: service auditd restart  

После данных изменений у нас начнут локально записываться логи аудита. Чтобы проверить, что
логи аудита начали записываться локально, нужно внести изменения в файл  

>/etc/nginx/nginx.conf  

или поменять его атрибут, потом посмотреть информацию об изменениях:  

> ausearch -f
> /etc/nginx/nginx.conf  

Также можно воспользоваться поиском по файлу  

> /var/log/audit/audit.log  

указав тэг _nginx_conf_:  

> grep nginx_conf /var/log/audit/audit.log

![](https://github.com/Vitaliy7/logs/blob/main/screenshots/audit1.png?raw=true)

 Теперь настроим пересылку логов на удаленный сервер. _Auditd_ по умолчанию не умеет пересылать логи,поэтому
для пересылки на сервере устанавливаем пакет _audispd-plugins_:

> yum -y install audispd-plugins  

Найдем и поменяем следующие строки в файле  

_/etc/audit/auditd.conf_:  

> log_format = RAW  
name_format = HOSTNAME    

В _name_format_ указываем _HOSTNAME_, чтобы в логах на удаленном сервере отображалось имя хоста.
В файле _/etc/audisp/plugins.d/au-remote.conf_ поменяем параметр _active_ на _yes_:
> active = yes  
direction = out  
path = /sbin/audisp-remote  
type = always  
#args =  
format = string  

В файле _/etc/audisp/audisp-remote.conf_ указываем адрес сервера и порт, на который будут
отправляться логи: 
>remote_server = 192.168.50.15  
port = 60
….  

Далее перезапускаем службу _auditd_:  
>service auditd restart  

На этом настройка web-сервера завершена. Теперь настраиваем _log_-сервер.
Откроем порт _TCP 60_, для этого уберем значки комментария в файле _/etc/audit/auditd.conf_:  

> tcp_listen_port = 60  

Перезапускаем службу _auditd_: 

> service auditd restart  

На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла 
_/etc/nginx/nginx.conf_ и проверить на log-сервере, что пришла информация об изменении атрибута:

![](https://github.com/Vitaliy7/logs/blob/main/screenshots/audit3.png?raw=true)
