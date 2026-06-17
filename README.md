# Rede Doméstica

Inventário e documentação da infraestrutura de rede doméstica (Fedora 44,
KVM/libvirt, mesh Wi-Fi Mercusys Halo H80X).

## Conteúdo

- [`CLAUDE.md`](CLAUDE.md) — contexto completo da topologia, lições
  aprendidas e decisões de configuração.
- [`inventory/dispositivos.csv`](inventory/dispositivos.csv) — tabela de
  dispositivos com IP fixo, MAC e método de reserva.
- [`docs/manuals/`](docs/manuals/) — manuais e datasheets oficiais da mesh
  Mercusys Halo H80X.

## Topologia (resumo)

```
Internet (fibra)
    │
    ▼
ZTE ZXHN F6645P (ONT da Claro, modo bridge)
    │
    ▼
Mesh Mercusys Halo H80X — Unidade SALA (router/DHCP/Wi-Fi)
    │
    └─ Backhaul cabeado ──► Mesh Halo H80X — Unidade ESCRITÓRIO
```

Detalhes completos em [`CLAUDE.md`](CLAUDE.md).

## Nota sobre o GPL do firmware

O código-fonte GPL liberado pelo fabricante para a Halo H80X não está neste
repositório (arquivo grande, redundante com o que o fabricante já publica).
Disponível diretamente no site da Mercusys, na seção de suporte/GPL do
produto Halo H80X.
