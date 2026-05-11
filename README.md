# Cisco CP-8945 + Issabel PBX — Guia de Configuração

> Guia completo para registrar o telefone **Cisco CP-8945** (firmware Enterprise SIP) no servidor **Issabel PBX** em um cenário com NAT e túnel WireGuard via Mikrotik.

---

## Variáveis usadas neste documento

Substitua pelos seus valores reais antes de aplicar:

| Variável | Descrição | Exemplo |
|---|---|---|
| `<PBX_IP>` | IP do servidor Issabel na rede do túnel | `10.x.x.x` |
| `<MIKROTIK_IP>` | IP do roteador Mikrotik (gateway NAT) | `10.x.x.1` |
| `<PHONE_LOCAL_IP>` | IP interno do telefone Cisco | `192.168.x.x` |
| `<WG_SERVER_IP>` | IP do servidor WireGuard (endpoint) | `x.x.x.x` |
| `<WG_SERVER_PORT>` | Porta UDP do WireGuard | `51820` |
| `<WG_NETWORK>` | Rede do túnel WireGuard | `10.x.x.0/24` |
| `<SIP_PORT>` | Porta SIP do Issabel | `5066` |
| `<RAMAL>` | Número do ramal | `1001` |
| `<SENHA>` | Senha do ramal | — |
| `<MAC>` | MAC do telefone (sem separadores, maiúsculas) | `AABBCCDDEEFF` |

---

## 1. Arquitetura da Solução

```
[Cisco CP-8945]──────[Mikrotik/NAT]══WireGuard══[Issabel PBX]
 192.168.x.x          <MIKROTIK_IP>               <PBX_IP>
 (rede local)         (gateway)                  (servidor SIP)
```

**Por que TCP em vez de UDP?**

O Mikrotik (mesmo com SIP ALG desativado) remapeia as portas UDP de origem a cada milissegundo, quebrando o handshake de registro SIP. O TCP cria uma conexão **persistente** que sobrevive ao NAT, resolvendo o problema de `401 Unauthorized` em loop.

---

## 2. Configuração do Túnel WireGuard (Mikrotik)

### 2.1 Criar a interface WireGuard no Mikrotik

```
/interface wireguard
add name=wg0 listen-port=<WG_SERVER_PORT> private-key="<CHAVE_PRIVADA_MIKROTIK>"
```

### 2.2 Adicionar o peer (servidor remoto)

```
/interface wireguard peers
add interface=wg0 \
    public-key="<CHAVE_PUBLICA_SERVIDOR>" \
    endpoint-address=<WG_SERVER_IP> \
    endpoint-port=<WG_SERVER_PORT> \
    allowed-address=<WG_NETWORK> \
    persistent-keepalive=25s
```

### 2.3 Adicionar o IP do túnel

```
/ip address
add address=<MIKROTIK_IP>/24 interface=wg0
```

### 2.4 Rota para o servidor PBX

```
/ip route
add dst-address=<PBX_IP>/32 gateway=wg0
```

### 2.5 Regra de NAT para o telefone acessar o PBX

```
/ip firewall nat
add chain=srcnat \
    src-address=<PHONE_LOCAL_IP> \
    dst-address=<PBX_IP> \
    action=masquerade \
    out-interface=wg0
```

### 2.6 Liberar tráfego SIP e RTP no firewall do Mikrotik

```
/ip firewall filter
add chain=forward src-address=<PHONE_LOCAL_IP> \
    dst-address=<PBX_IP> protocol=tcp dst-port=<SIP_PORT> action=accept

add chain=forward src-address=<PHONE_LOCAL_IP> \
    dst-address=<PBX_IP> protocol=udp dst-port=10000-30000 action=accept
```

> **⚠️ Importante:** Desabilitar o **SIP ALG** no Mikrotik se estiver ativo:
> ```
> /ip firewall service-port disable sip
> ```

---

## 3. Configuração do Servidor Issabel

### 3.1 Habilitar TCP no Asterisk

Edite `/etc/asterisk/sip_general_custom.conf`:

```ini
tcpenable=yes
tcpbindaddr=0.0.0.0:<SIP_PORT>
videosupport=yes
allow=h264
allow=h263
```

```bash
asterisk -rx "sip reload"
# Verificar que o TCP subiu:
netstat -tlnp | grep <SIP_PORT>
# Esperado: tcp  0.0.0.0:<SIP_PORT>  LISTEN  asterisk
```

### 3.2 Estender faixa de portas RTP (necessário para vídeo)

Edite `/etc/asterisk/rtp_custom.conf`:

```ini
[general]
rtpstart=10000
rtpend=30000
```

```bash
asterisk -rx "module reload res_rtp_asterisk.so"
```

### 3.3 Firewall (iptables)

```bash
iptables -I INPUT -p tcp --dport <SIP_PORT> -j ACCEPT
iptables -I INPUT -p udp --dport 10000:30000 -j ACCEPT
iptables -I OUTPUT -p udp --dport 10000:30000 -j ACCEPT
service iptables save
```

### 3.4 Fail2Ban — Whitelist do IP do Mikrotik

Edite `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 <MIKROTIK_IP> <WG_NETWORK>
```

```bash
systemctl restart fail2ban
```

---

## 4. Configuração dos Ramais no Issabel

**PBX > Configurações do PBX > Extensions > [ramal]**

Para **cada ramal** que vai usar um Cisco ou softphone com vídeo:

| Campo | Valor |
|---|---|
| **transport?** | `Todos - TCP Primario` |
| **nat?** | `Sim` |
| **disallow?** | `all` |
| **allow?** | `ulaw&alaw&gsm&g729&opus&h264&h263` |

Clique em **Submit** → **Apply Config**.

**Verificar:**
```bash
asterisk -rx "sip show peer <RAMAL>" | grep -E "Codec|Video|Transport"
# Esperado:
# Video Support: Yes
# Allowed.Trsp : UDP,TCP
# Codecs       : (ulaw|alaw|gsm|g729|opus|h264|h263)
```

---

## 5. Arquivo XML de Provisionamento TFTP

Salvo em `/tftpboot/SEP<MAC>.cnf.xml`

Use o template `config_templates/v9.2.2/SEP_TEMPLATE.cnf.xml` deste repositório.

### Campos obrigatórios a editar:

```xml
<!-- Servidor SIP -->
<processNodeName><PBX_IP></processNodeName>
<sipPort><SIP_PORT></sipPort>

<!-- Protocolo TCP (1=TCP | 2=UDP) -->
<transportLayerProtocol>1</transportLayerProtocol>

<!-- Data/Hora Brasil (São Paulo) -->
<dateTemplate>D/M/Y</dateTemplate>
<timeZone>SA Eastern Standard Time</timeZone>
<olsonTimeZone>America/Sao_Paulo</olsonTimeZone>

<!-- Ramal e credenciais -->
<line button="1" lineIndex="1">
  <proxy>USECALLMANAGER</proxy>
  <port><SIP_PORT></port>
  <name><RAMAL></name>
  <displayName><RAMAL> - Nome</displayName>
  <authName><RAMAL></authName>
  <authPassword><SENHA></authPassword>
</line>

<!-- Portas de mídia -->
<startMediaPort>16384</startMediaPort>
<stopMediaPort>32766</stopMediaPort>
<startVideoPort>20000</startVideoPort>
<stopVideoPort>30000</stopVideoPort>

<!-- Vídeo habilitado -->
<vendorConfig>
  <videoCapability>1</videoCapability>
  <cameraEnabled>true</cameraEnabled>
</vendorConfig>

<!-- Diretório de contatos -->
<directoryURL>http://<PBX_IP>/directory.php</directoryURL>
```

---

## 6. Diretório de Contatos (PHP)

Use o template `cisco_directory/directory.php.template` e renomeie para `directory.php`.

Edite o array `$contacts` com seus ramais e envie para o servidor:

```bash
scp directory.php root@<PBX_IP>:/var/www/html/
```

**Testar:** `http://<PBX_IP>/directory.php`

**No telefone:** botão Contatos → Diretório Corporativo

---

## 7. Adicionando um Novo Cisco CP-8945

1. Descubra o MAC (`Settings > Status > Network Information` no telefone).
2. Copie o template e renomeie:
```bash
cd /tftpboot
cp SEP_TEMPLATE.cnf.xml SEP<NOVO_MAC>.cnf.xml
nano SEP<NOVO_MAC>.cnf.xml
```
3. Edite: `<name>`, `<displayName>`, `<authName>`, `<authPassword>`, `<processNodeName>`.
4. Crie o ramal no Issabel (seção 4).
5. Reinicie o telefone.

---

## 8. Fluxo de Registro SIP (Diagnóstico)

```bash
asterisk -rvvvvv
sip set debug on
```

Sequência esperada no log:
```
TFTP: baixa SEP<MAC>.cnf.xml ..................... ✅
REGISTER via TCP → 401 Unauthorized .............. ✅ (normal)
REGISTER com Authorization (senha) → 200 OK ...... ✅
Telefone exibe ramal na tela ..................... ✅
```

---

## 9. Troubleshooting — Erros Comuns

| Erro | Causa | Solução |
|---|---|---|
| `401` em loop sem `200 OK` | NAT/UDP quebrando portas | Usar TCP no XML e no Issabel |
| `403 - Wrong password` | Senha errada | Corrigir `authPassword` no XML |
| `403 - Device not configured to use this transport` | Ramal só em UDP | Mudar para `Todos - TCP Primario` |
| `Unable to bind SIP TCP server` | Porta TCP já em uso | Verificar `tcpbindaddr` no `sip_general_custom.conf` |
| `RTP Read too short` | Pacotes de vídeo fragmentados | Reduzir bitrate do softphone |
| Fail2Ban bloqueando | IP do Mikrotik banido | `fail2ban-client unban <MIKROTIK_IP>` |
| `Serious Network Trouble` | Conexão TCP antiga morreu | Normal após reboot do telefone |
| `cm-closed-tcp` no alarm do Cisco | Asterisk não abriu TCP | `netstat -tlnp \| grep <SIP_PORT>` |

---

## 10. Estrutura do Repositório

```
.
├── README.md                          # Este arquivo
├── .gitignore                         # Exclui configs reais e firmware
├── cisco_directory/
│   └── directory.php.template         # Template do diretório de contatos
├── config_templates/
│   └── v9.2.2/
│       ├── SEP_TEMPLATE.cnf.xml       # Template XML do telefone
│       ├── XMLDefault.cnf.xml         # Config padrão TFTP
│       └── dialplan.xml               # Plano de discagem
└── docs/
    └── Cisco-7900-and-8800-series-freepbx-setup/  # Referência adicional
```

> **⚠️ Atenção:** Os arquivos de firmware (`.bin.sgn`, `.loads`) são proprietários da Cisco, mas estão neste repositório. Baixe-os diretamente da Cisco ou de fontes autorizadas por segurança.

---

## 11. Referências

- [Cisco CP-8945 Administration Guide (SIP)](https://www.cisco.com/c/en/us/td/docs/voice_ip_comm/cuipph/8941_8945/sip/8_0/english/administration/guide/)
- [Asterisk chan_sip documentation](https://wiki.asterisk.org/wiki/display/AST/Configuring+chan_sip)
- [Issabel PBX](https://www.issabel.org/)
- [MikroTik WireGuard](https://help.mikrotik.com/docs/display/ROS/WireGuard)
