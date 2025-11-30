# Deploy do Firefly III para JSJ (Ubuntu 22.04)

Guia resumido para publicar a instância "JSJ Finanças" em uma VPS Ubuntu 22.04 com Nginx, PHP 8.2 e MariaDB/MySQL.

## Pré-requisitos do servidor
- Ubuntu 22.04 atualizado (`sudo apt update && sudo apt upgrade`).
- Acesso sudo.
- DNS apontando para `jsjcorp.com.br` ou subdomínio (ex.: `financas.jsjcorp.com.br`).

## Pacotes base
```bash
sudo apt install -y nginx mariadb-server software-properties-common git unzip
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install -y php8.2 php8.2-fpm php8.2-cli php8.2-curl php8.2-xml php8.2-mbstring \
    php8.2-zip php8.2-bcmath php8.2-gd php8.2-intl php8.2-mysql php8.2-readline
```

## Banco de dados
```bash
sudo mysql_secure_installation
mysql -u root -p -e "CREATE DATABASE firefly CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER 'firefly'@'localhost' IDENTIFIED BY 'senha_forte';"
mysql -u root -p -e "GRANT ALL PRIVILEGES ON firefly.* TO 'firefly'@'localhost'; FLUSH PRIVILEGES;"
```

## Código-fonte
```bash
sudo mkdir -p /var/www/firefly-jsj
sudo chown $USER:$USER /var/www/firefly-jsj
cd /var/www/firefly-jsj
git clone <repo-url> .
cp .env.example.jsj .env
```
- Ajuste `APP_URL` no `.env` para o domínio escolhido (subdomínio `https://financas.jsjcorp.com.br` ou subpasta `https://jsjcorp.com.br/financas`).
- Preencha `APP_KEY` com `php artisan key:generate --show` e cole no `.env`.
- Configure credenciais do banco e SMTP (mantidos vazios por padrão).

## Dependências do app
```bash
cd /var/www/firefly-jsj
composer install --no-dev --optimize-autoloader
php artisan migrate --force
```

## Permissões de pasta
```bash
sudo chown -R www-data:www-data /var/www/firefly-jsj/storage /var/www/firefly-jsj/bootstrap/cache
sudo find /var/www/firefly-jsj/storage -type d -exec chmod 775 {} \;
sudo find /var/www/firefly-jsj/bootstrap/cache -type d -exec chmod 775 {} \;
```

## Configuração do PHP-FPM
- Certifique-se de que o socket esteja em `/run/php/php8.2-fpm.sock` (valor padrão).
- Ajuste `cgi.fix_pathinfo=0` em `/etc/php/8.2/fpm/php.ini` se necessário.
- Reinicie: `sudo systemctl restart php8.2-fpm`.

## Configuração do Nginx
Exemplo de vhost em `/etc/nginx/sites-available/firefly-jsj`:
```nginx
server {
    listen 80;
    server_name financas.jsjcorp.com.br;
    root /var/www/firefly-jsj/public;

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
```bash
sudo ln -s /etc/nginx/sites-available/firefly-jsj /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```
- Para usar subpasta (`jsjcorp.com.br/financas`), ajuste `server_name` para `jsjcorp.com.br` e adicione `location /financas { alias /var/www/firefly-jsj/public/; ... }` mantendo o `APP_URL` condizente.

## Queue/Job worker
- Firefly III utiliza jobs para importações e notificações. Recomenda-se `php artisan queue:work` gerenciado pelo systemd ou Supervisor.
- Exemplo de serviço systemd `/etc/systemd/system/firefly-queue.service`:
```
[Unit]
Description=Firefly Queue Worker
After=network.target

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/firefly-jsj/artisan queue:work --sleep=3 --tries=3 --timeout=120

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now firefly-queue.service
```

## Cron para tarefas agendadas
```bash
echo "* * * * * www-data cd /var/www/firefly-jsj && php artisan schedule:run >> /var/log/firefly-schedule.log 2>&1" | sudo tee /etc/cron.d/firefly-schedule
sudo chmod 644 /etc/cron.d/firefly-schedule
sudo systemctl restart cron
```

## HTTPS (Let’s Encrypt)
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d financas.jsjcorp.com.br # ou jsjcorp.com.br/financas com ajuste manual
sudo systemctl reload nginx
```

## Limpeza/considerações de produção
- Mantenha `APP_DEBUG=false`.
- Variáveis de SMTP estão vazias no `.env.example.jsj`; preencha e teste envio de e-mail.
- Remova ou desabilite pacotes/devtools não usados; não há arquivos de exemplo críticos a apagar.

## Checklist de Produção
- `APP_KEY` gerada e configurada.
- `APP_DEBUG=false`.
- `APP_URL` correta para o domínio/subpasta.
- Banco criado e migrado (`php artisan migrate --force`).
- Queue worker configurado (systemd/Supervisor) e cron ativo (`php artisan schedule:run`).
- HTTPS ativo (Certbot/Let’s Encrypt) e Nginx recarregado.
