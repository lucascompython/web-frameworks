{{#language.version}}
  FROM php:{{{.}}}-fpm
{{/language.version}}
{{^language.version}}
  FROM php:8.4-fpm
{{/language.version}}

RUN apt-get -qq update
RUN apt-get -qy install --no-install-recommends libzip-dev {{#deps}} {{{.}}} {{/deps}}

WORKDIR /usr/src/app

RUN docker-php-ext-install zip opcache

{{#environment}}
  ENV {{{.}}}
{{/environment}}

{{#build_deps}}
  RUN apt-get -qy install --no-install-recommends {{{.}}}
{{/build_deps}}

RUN apt-get clean && rm -rf /var/lib/apt/lists/*

{{#modules}}
  RUN docker-php-ext-install {{{.}}}
{{/modules}}

{{#extensions}}
  RUN mkdir -p /usr/src/php/ext/{{{.}}}
  RUN curl -fsSL https://pecl.php.net/get/{{{.}}} | tar xvz -C "/usr/src/php/ext/{{{.}}}" --strip 1
  RUN docker-php-ext-install {{{.}}}
{{/extensions}}

ENV EVENT_EXT_FILE /usr/local/etc/php/conf.d/docker-php-ext-event.ini
RUN if [ -f "${EVENT_EXT_FILE}" ] ; then \
  rm -fr /usr/local/etc/php/conf.d/docker-php-ext-sockets.ini ; \
  sed -i -e '1i extension=sockets.so' /usr/local/etc/php/conf.d/docker-php-ext-event.ini ; fi
  
{{#before_build}}
  RUN {{{.}}}
{{/before_build}}

{{#files}}
  COPY '{{source}}' '{{target}}'
{{/files}}

  RUN apt-get -qq update
  RUN apt-get -qy install curl


{{^framework.standalone}}
  {{#bootstrap}}
    RUN {{{.}}}
  {{/bootstrap}}
{{/framework.standalone}}

{{#command}}
  ENTRYPOINT {{{.}}}
{{/command}}

{{^command}}
  {{#slasheddocroot}}
    RUN sed -i 's/\;prefix.*/prefix = {{{.}}}/g' /usr/local/etc/php-fpm.d/www.conf
  {{/slasheddocroot}}
  {{^slasheddocroot}}
    RUN sed -i 's/\;prefix.*/prefix = \/usr\/src\/app\/public/g' /usr/local/etc/php-fpm.d/www.conf
  {{/slasheddocroot}}
  RUN sed -i 's/\(listen =\).*/\1 \/var\/run\/php-fpm.sock/g' /usr/local/etc/php-fpm.d/www.conf
  RUN sed -i 's/\;\(listen\.owner.*\).*/\1/g' /usr/local/etc/php-fpm.d/www.conf
  RUN sed -i 's/\;\(listen\.group.*\).*/\1/g' /usr/local/etc/php-fpm.d/www.conf
  RUN sed -i 's/\;\(listen\.mode.*\).*/\1/g' /usr/local/etc/php-fpm.d/www.conf
  # pm.max_children set according to nproc
  RUN if [ $(nproc) = 1 ] ; then sed -i 's/\(pm\.max_children =\).*/\1 512/g' /usr/local/etc/php-fpm.d/www.conf ; else sed -i 's/\(pm\.max_children =\).*/\1 1024/g' /usr/local/etc/php-fpm.d/www.conf ; fi 
  # after 15 seconds warm-up, half of the 'idle' children are not killed
  RUN if [ $(nproc) = 1 ] ; then sed -i 's/\(pm\.max_spare_servers =\).*/\1 256/g' /usr/local/etc/php-fpm.d/www.conf ; else sed -i 's/\(pm\.max_spare_servers =\).*/\1 512/g' /usr/local/etc/php-fpm.d/www.conf ; fi 
  
  RUN rm -fr /usr/local/etc/php-fpm.d/zz-docker.conf
  
  RUN echo 'server {\n\
      {{#docroot}}
        root {{{.}}};\n\
      {{/docroot}}
      {{^docroot}}
        root /usr/src/app/public;\n\
      {{/docroot}}
      listen 0.0.0.0:3000;\n\
      {{#nginx_conf}}
        {{{.}}}
      {{/nginx_conf}}
      {{^nginx_conf}}
        location / {\n\
          try_files $uri $uri/ /index.php;\n\
          fastcgi_pass unix:/var/run/php-fpm.sock;\n\
          fastcgi_param   SCRIPT_FILENAME         $document_root/index.php;\n\
          include fastcgi_params;\n\
        }\n\
      {{/nginx_conf}}
  }\n'\
  > /etc/nginx/conf.d/server.conf
  
  RUN echo 'opcache.enable=1\n\
  opcache.enable_cli=1\n\
  opcache.memory_consumption=512\n\
  opcache.interned_strings_buffer=64\n\
  opcache.max_accelerated_sources=32531\n\
  opcache.validate_timestamps=0\n\
  opcache.save_comments=1\n\
  opcache.fast_shutdown=0\n'\
  >> /usr/local/etc/php/conf.d/docker-php-ext-opcache.ini
  

  ENTRYPOINT /usr/local/sbin/php-fpm --daemonize; nginx -g "daemon off;"

{{/command}}

HEALTHCHECK CMD curl --fail http://0.0.0.0:3000 || exit 1
