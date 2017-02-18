#!/bin/bash
#Desabilitando o tráfego entre as placas
echo 0 > /proc/sys/net/ipv4/ip_forward
#Apagando e restaurando as chains e tabelas
iptables -Z # Zera as regras das chains
iptables -F # Remove as regras das chains
iptables -X # Apaga as chains
iptables -t nat -Z
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -Z
iptables -t mangle -F
iptables -t mangle -X

#Proteção contra ping, SYN Cookie, IP Spoofing e proteções do kernel
echo 1 > /proc/sys/net/ipv4/tcp_syncookies #Syn Flood-DoS
echo 1 > /proc/sys/net/ipv4/conf/all/rp_filter #Ip Spoofing
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts #Sem ping e port scanners
echo 0 > /proc/sys/net/ipv4/icmp_ignore_bogus_error_responses #Sem resposta remota
for i in /proc/sys/net/ipv4/conf/*; do
echo 0 > $i/accept_redirects #Sem redirecionar rotas
echo 0 > $i/accept_source_route #Sem traceroute
echo 1 > $i/log_martians #Loga pacotes suspeitos no kernel
echo 1 > $i/rp_filter #Ip Spoofing
echo 1 > $i/secure_redirects; done #Redirecionamento seguro de pacotes
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all #Sem ping e tracert
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
#Carregando os módulos
modprobe ip_tables
modprobe iptable_nat
modprobe iptable_filter
modprobe iptable_mangle
#Definindo políticas padrões
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
# desativa a resposta do comando ping
iptables -A OUTPUT -p icmp -o eth0 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type echo-reply -s 0/0 -i eth0 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type destination-unreachable -s 0/0 -i eth0 -j ACCEPT
iptables -A INPUT -p icmp --icmp-type time-exceeded -s 0/0 -i eth0 -j ACCEPT
iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -i eth0 -j DROP
#Liberando portas somente para a rede interna
iptables -A INPUT -p tcp --dport 3128 -i eth1 -j ACCEPT #Proxy
iptables -A INPUT -p tcp --dport 80 -i eth1 -j ACCEPT #HTTP
iptables -A INPUT -p tcp --dport 21 -i eth1 -j ACCEPT #FTP
iptables -A INPUT -p tcp --dport 53 -i eth1 -j ACCEPT #DNS
iptables -A INPUT -p udp --dport 53 -i eth1 -j ACCEPT #DNS
iptables -A INPUT -p tcp --dport 25 -i eth1 -j ACCEPT #SMTP
iptables -A INPUT -p tcp --dport 110 -i eth1 -j ACCEPT #SSL
iptables -A INPUT -p udp --dport 110 -i eth1 -j ACCEPT #SSL
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT #HTTP - Apache
iptables -A INPUT -p udp --dport 8080 -j ACCEPT #HTTP - Apache
# permite o SSH, FTP e portas passivas (configuradas em nosso guia)
iptables -A INPUT -p tcp --dport ssh --jump ACCEPT
iptables -A INPUT -p tcp --dport 20 --jump ACCEPT
iptables -A INPUT -p tcp --dport 21 --jump ACCEPT
iptables -A INPUT -p tcp --dport 60000:60005 --jump ACCEPT
iptables -A INPUT -s 127.0.0.1/32 --jump ACCEPT
# Habilita o roteamento no kernel #
echo 1 > /proc/sys/net/ipv4/ip_forward
# Compartilha a internet
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# nega qualquer conexão que não esteja na regra
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -L -n
