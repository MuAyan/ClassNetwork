## Jot Notes 
1. set hostname to mission-portal
2. new host configured at 127.0.1.1 as mission0portal.orion.local mission-portal in /etc/hosts
3. install and enable apache web server
4. create a fake portal page at /var/www/html/index.html for the attacker
5. create fake directories that show up in the directory listing
6. enable directory listing by editing "Options FollowSymLinks" to "Options Indexes FollowSymLinks" in /etc/apache2/apache2.conf and restart the server
7. set these in /etc/ssh/sshd_config to attract attackers:
Port 22
PermitRootLogin yes
PasswordAuthentication yes
Banner /etc/ssh/orion_banner
8. create a new banner in /etc/ssh/orion_banner/
*****************************************************
*   ORION RESEARCH STATION – MISSION CONTROL NODE   *
*   Unauthorized access is strictly prohibited.     *
*   All sessions are logged and monitored.          *
*****************************************************
9. create a decoy user: 
sudo useradd -m -s /bin/bash missionadmin
echo "missionadmin:orion2024" | sudo chpasswd
10. restart ssh
