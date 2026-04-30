## Notes 
1. set hostname to mission-portal
2. new host configured at 127.0.1.1 as mission0portal.orion.local mission-portal in /etc/hosts
3. install and enable apache web server
4. ```sudo chown -R -f www-data:www-data /var/www/html``` & ```sudo chmod -R 770 /var/www/html``` to give permissions to the apache web server files.
5. create a fake portal page at /var/www/html/index.html for the attacker
6. create fake directories that show up in the directory listing
7. enable directory listing by editing "Options FollowSymLinks" to "Options Indexes FollowSymLinks" in /etc/apache2/apache2.conf and restart the server
8. set these in /etc/ssh/sshd_config to attract attackers:
Port 22
PermitRootLogin yes
PasswordAuthentication yes
Banner /etc/ssh/orion_banner
9. create a new banner in /etc/ssh/orion_banner/
*****************************************************
*   ORION RESEARCH STATION – MISSION CONTROL NODE   *
*   Unauthorized access is strictly prohibited.     *
*   All sessions are logged and monitored.          *
*****************************************************
9. create a decoy user: 
sudo useradd -m -s /bin/bash missionadmin
echo "missionadmin:orion2024" | sudo chpasswd
10. restart apache2
11. sudo apt install vsftpd
12. sudo systemctl enable vsftpd
13.configure ```/etc/vsftpd.conf```
anonymous_enable=YES *Security concern, set because we need attackers to log in*
local_enable=YES
write_enable=NO
local_umask=022
anon_upload_enable=NO
anon_mkdir_write_enable=NO
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
xferlog_file=/var/log/vsftpd.log
listen=YES
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
pasv_min_port=60000
pasv_max_port=65000
log_ftp_protocol=YES
vsftpd_log_file=/var/log/vsftpd_full.log

14. restart ```systemctl restart vsftpd```
15. ```sudo ufw allow 21/tcp``` & ```sudo ufw allow 60000:65000/tcp``` to allow these ports through the firewalls
16. 
