Instituto Superior Técnico, Universidade de Lisboa

**Segurança Informática em Redes e Sistemas**

# Guia de Laboratório - *Virtual Private Network*

## Objectivo

O objectivo do guia consiste em aprender como funciona e como pode ser implementada uma VPN usando o software OpenVPN, que usa o protocolo TLS (Transport Layer Security) para proteger as comunicações.
Neste guia iremos ainda revisitar as ferramentas *nmap* e *iptables* (*firewall*).

---

## Exercício 1 - Restrição de acesso à rede

Obtenha o laboratório *Kathará* que servirá de base aos exercícios a desenvolver durante a aula.
A topologia é a apresentada na figura abaixo.
O lado esquerdo da rede (equipamentos ligados ao router 1) representa a rede de uma organização; o lado direito representa a Internet.

![][2]

1. Use o traceroute para verificar a que redes da organização consegue chegar a partir do PC1.
A quais não é possível?

2. Se controlasse o router 2 (mas não o router 1), conseguia fazer com o que PC1 tivesse acesso a todas as redes?
Se sim, faça-o.

3. Coloque regras de *firewall* no router 1 que impeçam o acesso de e à rede `192.168.0.0/24` a partir da Internet (que tem mais endereços do que os mostrados no diagrama).

4. No PC1, use o `nmap` para detetar as máquinas presentes na rede `200.200.200.128/25`.
Quantas máquinas foram encontradas e quais os serviços? (NB: o nmap vai percorrer cerca de 128 endereços, logo demora algum tempo)

```bash
nmap 200.200.200.128/25
```

5. Crie regras de *firewall* no router 1 que impeçam o acesso aos protocolos HTTP (80) e SSH (22) a partir da Internet, mas que permitam o acesso de dentro para fora (apenas para a rede `200.200.200.128/25` pois a `192.168.0.0/24` já se encontra bloqueada de e para a Internet).

6. Repita o comando `nmap` a partir do PC1 e do servidor 3 e verifique as diferenças.
Porque é que continuamos a conseguir aceder por ssh ao router 1 usando o IP `200.200.200.254`?

---

## Exercício 2 - Criação de uma VPN

Uma *Virtual Private Network* (VPN), ou rede privada virtual, permite o acesso a partir do exterior a uma rede interna de uma organização.
Adicionalmente permite proteger criptograficamente o tráfego que circula nas redes públicas.

Neste exercício vamos configurar o PC1 para aceder à rede interna (redes ligadas ao router 1), apesar do router 1 não permitir acessos do exterior. Portanto, o PC1 vai usar uma VPN para fazer esse acesso. Em concreto, vamos ter um cliente e um servidor OpenVPN:
* **Servidor OpenVPN:** o servidor designado simplesmente *VPN* (lado esquerdo, em baixo) vai ser o servidor OpenVPN, ou seja, o computador ao qual os computadores que queiram usar a VPN se terão de ligar.
* **Cliente OpenVPN:** o PC1 vai ser o cliente OpenVPN.

![image2](https://github.com/tecnico-sec/Kathara-VPN/assets/10196133/79146c8d-32f4-4bab-81f1-5b5b9b4c5ae0)

1. O OpenVPN usa chaves assimétricas e certificados para autenticar os utilizadores, logo precisa de uma *Public Key Infrastructure* (PKI) para fazer a criação, distribuição e revogação de certificados.
No entanto, para simplificar, não vamos usar uma PKI completa mas apenas uma PKI que inclui apenas uma *Certification Authority* (CA) muito simples. Aliás, a CA devia estar num ambiente com elevado grau de segurança, p.ex., num computador desligado da Internet, mas não vamos considerar esse aspecto.

O próprio OpenVPN já fornece um conjunto de *scripts* para gerar a CA e certificados: `easy-rsa`.
 
Na máquina VPN mude para a directoria `/etc/openvpn/` e execute:

```bash
cp -r /usr/share/easy-rsa /etc/openvpn/
cp /etc/openvpn/easy-rsa/vars.example /etc/openvpn/easy-rsa/vars
```

Edite o ficheiro `/etc/openvpn/easy-rsa/vars`.
Altere as últimas linhas para incluir o seguinte:

```
set_var KEY_COUNTRY "PORTUGAL"
set_var KEY_CITY "Lisbon"
set_var KEY_ORG "sirssec"
set_var KEY_EMAIL "admin@example.com"
set_var KEY_OU "OpenVPN"
```

Emita o seguinte comando na directoria `easy-rsa` para inicializar a PKI: 

```bash
./easyrsa init-pki
```

Emita o seguinte comando para inicializar a CA, criando o seu par de chaves e um certificado com a sua chave pública:

```bash
./easyrsa build-ca nopass
```

2. Temos agora as chaves da CA.
Vamos gerar chaves e certificados para o servidor e para o cliente continuando a usar os *scripts* `easy-rsa` (não defina *password*).

```bash
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-req client nopass
./easyrsa sign-req client client
```

3. Vamos gerar parâmetros *Diffie-Hellman* para o servidor usar quando clientes se tentarem ligar a ele:

```bash
./easyrsa gen-dh
```

4. O OpenVPN fornece um mecanismo chamado *tls-aut* que permite verificar a integridade das mensagens trocadas durante o *handshake* do TLS. Em geral não é possível garantir a integridade das mensagens trocadas *durante o handshake* do TLS, só *no fim do handshake*, pois até ao fim do *handshake* não existem chaves partilhadas que permitam verificar a integridade das mensagens. No TLS standard é isso que acontece: no fim do *handshake* cliente e servidor enviam um ao outro um MAC das mensagens trocadas e verificam se estão de acordo com o que esperavam.
O mecanismo *tls-aut* é um "truque" do OpenVPN que não funciona no caso geral mas por vezes pode funcionar: ter à partida uma chave secreta (simétrica) partilhada entre cliente e servidor para verificar a integridade das mensagens trocadas durante o *handshake* do TLS através da colocação de um MAC em cada mensagem. Este mecanismo serve para proteger de ataques de negação de serviço ou de tentativas de injecção de pacotes (p.ex. para tentar fazer buffer overflow). Gere essa chave que será partilhada entre o servidor e o(s) cliente(s):
```bash
openvpn --genkey --secret ta.key
cp ta.key /etc/openvpn
```

5. No PC1 copie o ficheiro exemplo de configuração para a área de *root*:

```bash
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~
```

6. Copie os seguintes ficheiros que foram gerados no servidor VPN (que estão na pasta `/etc/openvpn/easy-rsa/` e suas subpastas) para a área do *root* do PC1:
* `ca.crt` - certificado da CA
* `client.crt` - certificado do cliente
* `client.key` - chaves do cliente
* `ta.key` - a chave do mecanismo *tls-aut*

Sugestão: use o comando `scp` ou a pasta `shared`.

O ficheiro `client.key` contém a chave privada do cliente, logo tem de ser apagado do servidor. Esse ficheiro e o que contém a chave privada do servidor (`server.key`) têm de ser mantidos secretos. Escusado será dizer que seria muito errado copiar o ficheiro `server.key` para o cliente!


7. No PC1, edite o ficheiro `client.conf` no cliente e defina o endereço do servidor VPN (campo *remote*) e os nomes dos ficheiros com as chaves

8. No servidor de VPN, copie os ficheiros `ca.crt`, `dh.pem`, `server.key`, `server.crt` e `ta.key` para a pasta `/etc/openvpn/`.

9. Analise o ficheiro `server.conf` que já está na mesma pasta. Altere o seguinte:
* Mude `dh1024.pem` para `dh.pem`.
* Descomente a linha `tls-auth ta.key 0` (tire o ;).

10. Repare que o mesmo ficheiro contém a linha:

```bash
server 200.200.200.0 255.255.255.128
```

Esta linha indica que a subrede da VPN é a `200.200.200.0/25`, ou seja, que um computador externo (no nosso caso, o PC1) vai aparecer para a rede interna com um endereço IP dessa subrede.
Muito provavelmente, na sua configuração a comunicação entre essa subrede e a subrede `200.200.200.128/25` (que é parecida mas diferente) está barrada na firewall do router1. Remova essa restrição, reconfigurando a firewall.

11. Lance o servidor de VPN no servidor, em modo *debug*, a partir da pasta `/etc/openvpn`:

```bash
openvpn server.conf
```

Caso o openvpn falhe, poderá ser necessário correr o seguinte comando antes:
```
mkdir -p /dev/net
mknod /dev/net/tun c 10 200
chmod 600 /dev/net/tun
```

> (No futuro, para lançar normalmente use o script `/etc/init.d/openvpn`)

12. Descomente a seguinte linha ao ficheiro `client.conf` para activar a compressão das comunicações: `comp-lzo`.

13. Lance o cliente no PC1, em modo *debug*:

```bash
openvpn client.conf
```

14. Observe as mensagens no cliente e servidor.
Verifique que já estão ligados.

15. Verifique as rotas presentes no router 1 executando `ip route`.
Repare que existe já uma rota para a rede `200.200.200.0/25` através do servidor de VPN.
Esta é a rede com os IPs atribuídos aos clientes de VPN.

16. Coloque o openvpn em background com `Ctrl + z` seguido do comando `bg`.
Tente agora aceder do PC1 à rede que está bloqueada: `192.168.0.0/24`.

17. Faça `traceroute` do PC1 para o servidor 4.
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
  [5]: https://openvpn.net/community-resources/how-to/
