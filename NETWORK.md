# Contexto: Rede Doméstica do Ronaldo

Este arquivo dá contexto ao Claude Code sobre a infraestrutura de rede doméstica
do Ronaldo (Fedora 44, KVM/libvirt, mesh Wi-Fi). Mantenha atualizado conforme a
rede mudar.

## Visão geral da topologia

```
Internet (fibra)
    │
    ▼
ZTE ZXHN F6645P (ONT da Claro)
    │  [modo BRIDGE — sem Wi-Fi, sem DHCP, apenas conversor óptico→Ethernet]
    ▼
Mesh Mercusys Halo H80X — Unidade SALA (router/DHCP/Wi-Fi principal)
    │
    ├─ Porta 1: ZTE (uplink/WAN)
    ├─ Porta 2: OrangePi PC (cabeado)
    ├─ Porta 3: Backhaul cabeado ──────────► Mesh Halo H80X — Unidade ESCRITÓRIO
    │                                              ├─ Porta 1: Backhaul (recebido)
    │                                              ├─ Porta 2: Laptop Fedora (cabeado)
    │                                              └─ Porta 3: spare/livre
    │
    └─ Wi-Fi: celulares (Ronaldo, Juliana, Roberto), TV da sala,
              2x Chromecast, Xbox (em teste — pode migrar para cabo
              via switch Gigabit adicional se a performance não for boa)
```

### Notas da topologia

- **ZTE F6645P**: ONT da Claro. Será colocado em **modo bridge** (função oficial
  da Claro, sem necessidade de desbloqueio/hack). Em bridge, perde Wi-Fi e DHCP
  próprios; só repassa o sinal da fibra como Ethernet puro por uma única porta.
- **Mesh Mercusys Halo H80X** (kit com 2 unidades, Wi-Fi 6 AX3000, 3 portas
  Gigabit cada): assume 100% do roteamento, DHCP e Wi-Fi da casa após o bridge
  do ZTE. Tem **Address Reservation** nativo (DHCP Binding via app, em
  More → Advanced → Address Reservation) — diferente do ZTE da Claro, que não
  expõe essa opção na interface web.
- **Backhaul cabeado** entre as duas unidades mesh (cabo já existente entre
  sala e escritório) — preferível ao backhaul wireless padrão, pois libera o
  espectro de rádio das duas unidades para os clientes em vez de gastar banda
  com a comunicação entre nós.
- **Xbox**: testando primeiro via Wi-Fi (Wi-Fi 6, deve ser suficiente). Se a
  performance/latência não for boa, Ronaldo cogita comprar um **switch
  Gigabit não-gerenciado** para a sala, já que as 3 portas da unidade da sala
  estarão todas ocupadas (ZTE + OrangePi + backhaul). O switch permitiria
  conectar Xbox, TV e eventualmente um Wii, tudo cabeado.
- **TV da sala**: permanece no Wi-Fi.

## Inventário de dispositivos (IPs e MACs)

### Estado atual (antes da mesh — IPs fixos via `nmcli` manual)

O ZTE F6645P da Claro **não expõe DHCP Binding na interface web** (limitação
do firmware customizado pela operadora). Por isso, os dispositivos críticos
têm IP fixo configurado manualmente no próprio SO (`nmcli ... ipv4.method
manual`), fora da faixa DHCP dinâmica do roteador (`192.168.0.2`–`.50`).

| Dispositivo            | MAC                 | IP fixo         | Método                          |
|-------------------------|----------------------|------------------|----------------------------------|
| fedora (Ethernet, `br0`) | `30:24:A9:FB:C7:6F`  | `192.168.0.60`   | `nmcli` manual (interface `br0`) |
| fedora (Wi-Fi)           | `22:90:26:F9:0D:6F`  | `192.168.0.64`   | `nmcli` manual (`CLARO_F76183`)  |
| orangepipc (`end0`)      | `02:81:49:3B:6E:5D`  | `192.168.0.61`   | `nmcli` manual (`Wired connection 1`) |
| win11 (VM, KVM bridge)   | `52:54:00:D9:C3:41`  | `192.168.0.62`   | IP estático configurado no Windows |
| sandbox (VM, KVM bridge) | `52:54:00:F5:4F:0E`  | `192.168.0.63`   | `nmcli` manual (`Wired connection 1`) |

Gateway: `192.168.0.1` (ZTE). DNS: `1.1.1.1` / `1.0.0.1` (Cloudflare).
Faixa DHCP dinâmica do ZTE: `192.168.0.2`–`192.168.0.50` (lease time 12h /
43200s) — usada por celulares, Chromecasts, Xbox, TV, etc.

### Plano para quando a mesh entrar

- Migrar as reservas de IP para o **Address Reservation nativo da mesh**
  (DHCP Binding de verdade, via MAC).
- Possivelmente reverter `fedora` e `orangepi` de IP manual (`nmcli`) para
  DHCP automático, já que a reserva da mesh garante o mesmo IP de qualquer
  forma — mais simples de manter.
- `win11` e `sandbox` (VMs no host Fedora, via bridge `br0` do libvirt)
  também podem ser migradas para reserva via mesh.

## Outras infraestruturas relevantes

### VMs no host Fedora (KVM/libvirt)

- Bridge `br0` no host Fedora, ligada à interface física `enp1s0`
  (substituiu a rede NAT `default`/`virbr0` do libvirt).
- `win11`: VM Windows 11, interface `type='bridge'` em `br0`, modelo
  `e1000e`.
- `sandbox`: VM Linux (Fedora Server), interface `type='bridge'` em `br0`,
  modelo `virtio`. Tem Docker e Tailscale instalados.
- Funções Fish criadas para gerenciar as VMs sem digitar `virsh` diretamente:
  `win11start`, `win11stop`, `win11kill`, `sandboxstart`, `sandboxstop`,
  `sandboxkill` (em `~/.config/fish/functions/`). `stop` = shutdown gracioso;
  `kill` = `virsh destroy` (força bruta, só usar se a VM estiver travada).

### Resolução de nomes (`/etc/hosts` no host Fedora)

mDNS (Avahi) entre Fedora e OrangePi **não funciona de forma confiável**
nessa rede (diagnosticado: a query mDNS chega ao OrangePi via tcpdump, mas o
Avahi dele não responde — causa raiz não 100% confirmada, suspeita de
dependência anterior do Tailscale para essa resolução). Solução adotada:
entradas estáticas em `/etc/hosts`:

```
192.168.0.60   fedora
192.168.0.61   orangepi
192.168.0.62   win11
192.168.0.63   sandbox
137.131.145.213   oracle
62.84.180.13   contabo
```

`win11` resolve nome via **NetBIOS/Samba** (`nmblookup`) como alternativa —
funciona nativamente pois é Windows. Para isso ser possível, foi necessário
liberar ICMP no Firewall do Windows Defender dentro da VM (perfil de rede
"Público" bloqueia ping por padrão).

### Servidores remotos (fora da LAN)

| Nome      | IP               | Observação                                              |
|-----------|------------------|-----------------------------------------------------------|
| oracle    | `137.131.145.213` | Responde ping normalmente.                                |
| contabo   | `62.84.180.13`    | Ping bloqueado por padrão pelo **Contabo Firewall** (firewall de borda, separado do `ufw` interno da VPS). Foi liberado via painel de controle da Contabo (regra de Inbound ICMP). |

### Firewall do host Fedora (`firewalld`)

- Zona ativa: `FedoraWorkstation`, aplicada às interfaces `wlp0s20f3`, `br0`,
  `enp1s0`.
- Serviço `mdns` foi adicionado a essa zona (`firewall-cmd --add-service=mdns
  --permanent`) durante o diagnóstico de mDNS — mantido, mesmo que o mDNS
  para o OrangePi não tenha funcionado por outro motivo.
- Outros serviços liberados na zona: `dhcpv6-client`, `samba`,
  `samba-client`, `ssh`.

## Coisas a evitar / lições aprendidas

- **Não usar `virsh edit` sem `EDITOR=nano`** — o padrão é `vim`, que o
  Ronaldo não gosta de usar.
- **STP da bridge `br0` demora a estabilizar** (`listening → learning →
  forwarding`, ~15s por estágio) após qualquer mudança de configuração via
  `nmcli`. Um estado momentâneo de `NO-CARRIER`/`state DOWN` logo após mudar
  a bridge não é necessariamente um erro — pode só ser o STP em progresso.
  Verificar com `bridge link show` antes de assumir que algo quebrou.
- **Tailscale**: removido do Fedora (host). Ainda presente no OrangePi e na
  VM `sandbox`. Avaliar se vale remover dos dois também por consistência.
- Pretende-se criar um **repositório GitHub** para versionar este inventário
  (provavelmente como `README.md` + um CSV com a tabela de dispositivos),
  no lugar de (ou complementando) este arquivo.
