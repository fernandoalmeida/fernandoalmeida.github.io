---
layout: post
title: "Como usar Postgres local sem senha"
date: 2015-01-04 23:14:05 -0200
comments: true
meta: false
categories: [Dicas, Postgres]
---
Em ambiente de desenvolvimento podemos facilitar o uso do Postgres permitindo login sem senha.<br/>
Mostrarei aqui como fazer isso de forma simples e rapida.
<!--more-->

Para isto basta substituir o conteúdo do arquivo `/etc/postgresql/POSTGRES_VERSION/main/pg_hba.conf` por

    local  all  all                trust
    host   all  all  127.0.0.1/32  trust
    host   all  all  ::1/128       trust

Depois de reiniciar o serviço já será permitida conexão, tanto local quanto via rede/VM, com qualquer usuário (inclusive o `postgres`) sem senha.

Para testar execute o comando `psql -U postgres`.

#Referência:
http://www.postgresql.org/docs/9.4/static/auth-pg-hba-conf.html
