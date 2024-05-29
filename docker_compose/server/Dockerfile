# Utiliser l'image officielle httpd comme image de base
FROM httpd:2.4
 
# Copier index.html dans le répertoire de contenu web de Apache
COPY index.html /usr/local/apache2/htdocs/
 
COPY httpd.conf /usr/local/apache2/conf/httpd.conf