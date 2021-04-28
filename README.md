Instituto Superior Técnico, Universidade de Lisboa

**Segurança Informática em Redes e Sistemas**

# Guia de Laboratório - *Virtual Private Network*

## Objectivo

O objectivo do guia consiste em aprender como funciona e como pode ser implementada uma VPN usando o software OpenVPN.
Neste guia iremos ainda revisitar as ferramentas *nmap* e *iptables* (*firewall*).

---

## Exercício 1 -- Restrição de acesso à rede

Obtenha o laboratório *Kathará* que servirá de base aos exercícios a desenvolver durante a aula.
A topologia é a apresentada na figura abaixo.
O lado esquerdo da rede (equipamentos ligados ao router 1) representa a rede de uma organização; o lado direito representa a Internet.

![][2]

1. Use o traceroute para verificar a que redes da organização consegue chegar a partir do PC1.
A quais não é possível?

2. Se controlasse o router 2 (mas não o router 1), conseguia fazer com o que PC1 tivesse acesso a todas as redes?
Se sim, faça-o.

3. Coloque regras de *firewall* no router 1 que impeçam o acesso de e à rede `192.168.0.0/24` a partir da Internet (que tem mais endereços do que os utilizados).

4. No PC1, use o `nmap` para detetar as máquinas presentes na rede `200.200.200.128/25`.
Quantas máquinas foram encontradas e quais os serviços?

```bash
nmap 200.200.200.128/25
```

5. Crie regras de *firewall* no router 1 que impeçam o acesso aos protocolos HTTP (80) e SSH (22) a partir da Internet, mas que permitam o acesso de dentro para fora (apenas para a rede `200.200.200.128/25` pois a `192.168.0.0/24` já se encontra bloqueada de e para a Internet).

6. Repita o comando `nmap` a partir do PC1 e do servidor 3 e verifique as diferenças.
Porque é que continuamos a conseguir aceder por ssh ao router 1 usando o IP `200.200.200.254`?

---

## Exercício 2 -- Criação de uma VPN

Uma *Virtual Private Network* (VPN) permite o acesso a partir do exterior a uma rede interna de uma organização.
Adicionalmente permite garantir que o tráfego que circula nas redes públicas é cifrado.
Neste exercício vamos configurar o PC1 para aceder à rede interna (redes ligadas ao router 1), apesar do router 1 não permitir acessos do exterior.
O servidor VPN vai ser usado para terminar a VPN.

1. O OpenVPN usa chaves assimétricas e certificados para autenticar os utilizadores.
Em vez de montarmos uma PKI completa, vamos usar um conjunto de *scripts* fornecidos pelo OpenVPN para gerar uma CA e certificados.
Na máquina VPN mude para a directoria `/etc/openvpn/`, corra `cp -r /usr/share/easy-rsa /etc/openvpn/` e `cp /etc/openvpn/easy-rsa/vars.example /etc/openvpn/easy-rsa/vars`.
Edite o ficheiro `/etc/openvpn/easy-rsa/vars`.
Altere as últimas linhas para incluir o seguinte:

```
set_var KEY_COUNTRY "PORTUGAL"
set_var KEY_CITY "Lisbon"
set_var KEY_ORG "sirssec"
set_var KEY_EMAIL "admin@example.com"
set_var KEY_OU "OpenVPN"
```

Emita os seguintes comandos para criar as chaves e certificado da CA.

```bash
./easyrsa init-pki
./easyrsa build-ca nopass
```

2. Temos agora as chaves da CA.
Vamos gerar chaves e certificados para o servidor e um cliente (não defina *password*).

```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-req client nopass
./easyrsa sign-req client client
```

3. Por fim geramos os parâmetros *Diffie-Hellman* para uso pelo servidor.

```bash
./easyrsa gen-dh
```

4. No PC1 copie o ficheiro exemplo de configuração para a área de *root*:

```bash
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~
```

5. Copie os seguintes ficheiros que foram gerados no servidor VPN (estão na pasta `/etc/openvpn/easy-rsa/pki/`) para a área do *root* do PC1: `ca.crt`, `client.crt`, `client.key` e `ta.key`.
Os ficheiros do cliente (principalmente as chaves) devem ser apagadas do servidor e mantidas secretas.
Sugestão: use o comando `scp` ou a pasta shared.

6. No PC1, edite o ficheiro `client.conf` no cliente e defina o endereço do servidor VPN (campo *remote*) e os nomes dos ficheiros com as chaves

7. No servidor de VPN, copie os ficheiros gerados (`ca.crt` `dh.pem` `server.key` `server.crt` `ta.key`) para a pasta `/etc/openvpn/`.

8. Analise o ficheiro `server.conf` que já está na mesma pasta. 
Mude nesse ficheiro `dh1024.pem` para `dh.pem`.
Descomente a linha `tls-auth ta.key 0` (tire o ;).

9. Lance o servidor de VPN no servidor, em modo *debug*, a partir da pasta `/etc/openvpn`:

```bash
openvpn server.conf
```

Caso o openvpn falhe, poderá ser necessario correr o seguinte comando antes:
```
mkdir -p /dev/net
mknod /dev/net/tun c 10 200
chmod 600 /dev/net/tun
```

> (No futuro, para lançar normalmente use o script `/etc/init.d/openvpn`)

10. Adicione a seguinte linha ao ficheiro client.conf:
```
comp-lzo
```


11. Lance o cliente no PC1, em modo *debug*:

```bash
openvpn client.conf
```

12. Observe as mensagens no cliente e servidor.
Verifique que já estão ligados.

13. Verifique as rotas presentes no router 1.
Repare que existe já uma rota para a rede `200.200.200.0/25` através do servidor de VPN.
Esta é a rede com os IPs atribuídos aos clientes de VPN.

15. Coloque o openvpn em background com `Ctrl + z` seguido do comando `bg`.
Tente agora aceder do PC1 à rede que está bloqueada: `192.168.0.0/24`.

15. Faça `traceroute` do PC1 para o servidor 4.
O router 2 está presente no caminho?
Porquê?

---

Referências:

-   *Kathará*, [https://github.com/KatharaFramework/Kathara/wiki][3]

-   Oskar Andreasson, Iptables Tutorial, version 1.2.2,
    [http://www.frozentux.net/iptables-tutorial/iptables-tutorial.html][4],
    2006

-   OpenVPN Howto,
    [https://openvpn.net/community-resources/how-to/][5]


  [2]: media/image2.png 
  [3]: https://github.com/KatharaFramework/Kathara/wiki
  [4]: http://www.frozentux.net/iptables-tutorial/iptables-tutorial.html
  [5]: [https://openvpn.net/community-resources/how-to/]
