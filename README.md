# Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux

Projeto educacional da DIO sobre auditoria de autenticação em ambiente controlado.

> **Escopo executado:** o sandbox não possui VirtualBox, Kali Linux ou Metasploitable 2. Como o desafio permite adaptação, foi executada uma variante real e reproduzível em `127.0.0.1`, com FTP/vsftpd, Samba e um formulário web sintético inspirado no DVWA. Nenhum host externo foi testado.

## Objetivos

- Demonstrar descoberta controlada de serviços.
- Executar uma verificação de credenciais de laboratório com Medusa em FTP.
- Enumerar e validar um compartilhamento SMB com `smbclient`.
- Testar o comportamento de um formulário web local.
- Registrar limitações da ferramenta e recomendar mitigações.

## Laboratório executado

| Componente | Implementação | Escopo |
|---|---|---|
| Estação de auditoria | Ubuntu do sandbox com Medusa 2.2 e Nmap 7.94 | local |
| Serviço FTP | vsftpd 3.0.5 em `127.0.0.1:2121` | local |
| Serviço SMB | Samba 4.19 em `127.0.0.1:1445` | local |
| Formulário web | servidor Python sintético em `127.0.0.1:8088` | local |

As portas foram vinculadas a loopback. O serviço web usa somente contas sintéticas criadas para este teste. Senhas não são publicadas neste repositório.

## Preparação

Ferramentas instaladas no sandbox:

```bash
sudo apt install medusa nmap vsftpd samba smbclient
```

O laboratório foi executado exclusivamente em `127.0.0.1`. A descoberta foi:

```bash
nmap -sV -Pn -p 2121,1445,8088 127.0.0.1
```

O resultado confirmou:

- `2121/tcp`: vsftpd 3.0.5;
- `1445/tcp`: Samba smbd;
- `8088/tcp`: servidor HTTP Python.

A saída integral está em [`evidence/scan.txt`](evidence/scan.txt).

## 1. FTP com Medusa

Foi usado um conjunto mínimo de usuários e senhas sintéticos, com uma única tarefa concorrente:

```bash
medusa -h 127.0.0.1 -n 2121 \
  -U users.txt -P passwords.txt -M ftp -t 1 -f -v 6
```

Resultado real: o módulo FTP identificou a conta de laboratório autorizada e retornou `ACCOUNT FOUND`/`SUCCESS`. A evidência sanitizada está em [`evidence/medusa-ftp-single.txt`](evidence/medusa-ftp-single.txt).

**Risco demonstrado:** senha fraca ou reutilizada em serviço FTP legado pode ser descoberta por tentativas automatizadas.

## 2. SMB e password spraying

A enumeração autenticada do compartilhamento foi validada com:

```bash
smbclient -p 1445 -L //127.0.0.1/ -U '<usuario-de-lab>%<senha-de-lab>'
```

O compartilhamento `labshare` e o serviço `IPC$` foram listados. SMBv1 apareceu desabilitado.

O módulo `smbnt` do Medusa, nesta instalação, retornou mensagens de erro de modo de segurança e resultados `UNKNOWN_ERROR_CODE`; por isso esses achados foram tratados como **inconclusivos**, não como sucesso. A evidência está em [`evidence/smb.txt`](evidence/smb.txt).

**Risco demonstrado:** a enumeração e a autenticação SMB devem ser restringidas por segmentação, firewall e políticas de conta.

## 3. Formulário web local

O formulário local aceitou `POST /login` e retornou mensagens distintas para senha inválida e válida. Validação direta:

```bash
curl --data 'username=<usuario>&password=<senha-incorreta>' \\
  http://127.0.0.1:8088/login

curl --data 'username=<usuario>&password=<senha-de-lab>' \\
  http://127.0.0.1:8088/login
```

O teste manual retornou `Login incorrect` para a tentativa inválida e `Welcome lab user` para a credencial sintética correta.

O módulo `web-form` do Medusa 2.2 sofreu **segmentation fault** contra o formulário local. Isso foi registrado como limitação da ferramenta/combinação de parâmetros; nenhum resultado foi inventado. A evidência está em [`evidence/medusa-web.txt`](evidence/medusa-web.txt).

> Esta etapa é uma variante local do cenário DVWA. Não foi feito teste contra DVWA público ou aplicação de terceiros.

## Resultados

| Cenário | Resultado observado | Conclusão |
|---|---|---|
| Descoberta | 3 portas locais identificadas | escopo confirmado |
| FTP + Medusa | `ACCOUNT FOUND`/`SUCCESS` em conta sintética | teste reproduzido |
| SMB + smbclient | `labshare` e `IPC$` listados | enumeração reproduzida |
| SMB + Medusa | `UNKNOWN_ERROR_CODE` | inconclusivo; não tratar como sucesso |
| Web manual | respostas distintas para inválida/válida | comportamento reproduzido |
| Web + Medusa | segmentation fault no módulo 2.2 | limitação documentada |

## Mitigações recomendadas

1. Desabilitar FTP ou migrar para SFTP/FTPS.
2. Aplicar MFA e bloquear senhas comuns/reutilizadas.
3. Implementar rate limiting, backoff e bloqueio progressivo.
4. Evitar mensagens que revelem qual fator de autenticação falhou.
5. Restringir SMB por firewall e segmentação; manter SMBv1 desabilitado.
6. Monitorar tentativas por conta, origem e dispositivo.
7. Usar Argon2id/bcrypt com sal na aplicação web.
8. Repetir DAST/SAST e testes de regressão após a correção.
9. Atualizar o Medusa e validar módulos em uma cópia descartável antes de usar em auditorias autorizadas.

## Limitações e transparência

- Não foram utilizadas VMs Kali Linux/Metasploitable 2 por indisponibilidade no sandbox.
- O cenário web é sintético e não é uma instalação DVWA.
- O SMB foi validado com `smbclient`; o resultado do módulo Medusa foi inconclusivo.
- O módulo web do Medusa apresentou falha de segmentação; o teste manual foi usado apenas para confirmar o comportamento do formulário.
- Não há capturas de tela de uma VM real nem resultados de alvos externos.

## Estrutura

```text
README.md
users.txt
passwords.txt
evidence/scan.txt
evidence/medusa-ftp.txt
evidence/medusa-ftp-single.txt
evidence/medusa-web.txt
evidence/smb.txt
scripts/README.md
```

## Referências

- [Kali Linux](https://www.kali.org/)
- [DVWA](http://www.dvwa.co.uk/)
- [Medusa](http://www.foofus.net/jmk/medusa/medusa.html)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [DIO — projeto do curso](https://web.dio.me/)
