# Arx Painel

Painel web de administração do **Arx OS** — distribuição Linux baseada em Debian, focada em segurança e privacidade, para uso acadêmico e empresarial.

Versão atual: **0.36.0**

## O que é

Interface web completa para administrar um servidor Arx OS: rede, firewall, DNS, DHCP, domínio (Samba AD), usuários e grupos, backups, atualizações, segurança e muito mais — tudo sem precisar tocar em terminal para o dia a dia.

## Funcionalidades

- **Rede** — interfaces, IP, VLAN (criação/remoção com rollback automático)
- **Firewall** — regras legíveis, contadores, rollback automático
- **DHCP** e **DNS** (zonas BIND + integração Samba/AD, métricas de consultas)
- **Domínio** — administração completa de Samba AD (usuários, grupos, OUs)
- **Usuários e grupos locais**
- **Atualizações** — streaming ao vivo, isolamento via `systemd-run`
- **Backup e restauração**
- **Alertas e notificações** (e-mail / webhook)
- **API REST** (token Bearer, reaproveita todas as ações do painel)
- **Segurança**: RBAC (admin/leitura), MFA (TOTP), WebAuthn (chave física/biometria), AppArmor em modo enforce
- **Diagnóstico, saúde do sistema, auditoria, jobs assíncronos, modo manutenção**

## Requisitos

- Debian 13 (trixie) ou Arx OS — **não recomendado** em Ubuntu, Mint ou outros derivados sem testes adicionais
- Python 3 (dependências isoladas em `venv`)

## Instalação

```bash
wget http://arxos.is-a.dev/arx-archive-keyring.deb
sudo dpkg -i arx-archive-keyring.deb

echo "deb http://arxos.is-a.dev/repo stable main" | sudo tee /etc/apt/sources.list.d/arxos.list

sudo apt update
sudo apt install arx-painel
```

> O repositório público ainda está em preparação. Enquanto isso, builds `.deb` podem ser solicitados diretamente.

## Status do roadmap (Fase 5 — Robustez e Segurança)

- [x] RBAC v1
- [x] MFA (TOTP)
- [x] Notificações (e-mail/webhook)
- [x] WebAuthn
- [x] API completa
- [ ] Plugins *(adiado conscientemente para pós-1.0)*

Roadmap detalhado em `docs/Arx_Painel_Roadmap_Robustez_Seguranca_Profissional.md`.

## Arquitetura

O painel roda em duas camadas: **webui** (Flask, interface HTTP) e **helper** (processo privilegiado, comunicação via socket Unix) — separação que isola a interface web de qualquer ação sensível no sistema.

## Licença

[GNU AGPLv3](LICENSE)

## Contato

arxos.project@gmail.com
