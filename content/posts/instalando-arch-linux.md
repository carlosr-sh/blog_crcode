+++
date = '2026-01-15T16:31:56-03:00'
draft = false
title = 'Instalando Arch Linux'
+++

### Instalando Arch linux com encriptação luks, salvando a chave com TPM e secure boot

Estava eu querendo solucionar uma brecha nas minhas configurações do meu desktop, sempre tive a paranoia que um dia alguém pode ter acesso ao meu disco, meu dever é não facilitar esse acesso. 

Então depois de anos usando linux e [Archlinux](https://archlinux.org/), resolvi buscar a fundo e pesquisar sobre uma forma inteligente e segura de criptografar meu disco, e não ter que digitar a senha sempre que reiniciar meu computador.

Antes de mais nada eu estava testando tudo em uma maquina virtual para depois colocar em pratica no meu computador pessoal. Usarei o [Arch wiki](https://wiki.archlinux.org/title/Main_page) como base para todos os passos. 

Muitos tutoriais tratam o Secure boot como apenas "aquela coisa chata que impede o Linux de bootar". Aqui vamos entender o que ele é e como ele se comporta. 

O Conceito:

O Secure Boot é um protocolo da especificação UEFI. Ele impede a execução de binários (bootloaders, kernels, drivers) que não estejam assinados digitalmente por uma chave confiável.

A Hierarquia de Chaves (Key Hierarchy):

1. **PK (Platform Key):** A chave mestre. Geralmente controlada pelo fabricante do hardware. Quem tem a PK é o "dono" da máquina.
2. **KEK (Key Exchange Keys):** Chaves autorizadas a atualizar o banco de dados de assinaturas.
3. **db (Signature Database):** A lista branca. Contém os certificados ou hashes dos binários permitidos (como o bootloader do Windows ou o shim do Fedora/Ubuntu).
4. **dbx (Forbidden Signature Database):** A lista negra. Binários revogados (com falhas de segurança conhecidas).

O Cenário Arch Linux:

Diferente do Ubuntu ou Fedora, o kernel do Arch não é assinado pela Microsoft (que possui uma chave na db da maioria das placas-mãe).

**O Desafio:** Para usar Secure Boot no Arch, você tem duas opções: usar um "shim" assinado ou criar suas próprias chaves (PK, KEK, db), apagar as chaves do fabricante e assumir a posse da máquina.
  
**A Ferramenta:** Vamos usar o `sbctl` (Secure Boot Control). Ele facilita a criação de chaves personalizadas e a assinatura dos seus arquivos de boot.

Introduzindo 
```bash
sbctl status
```

Ele retorna:

```bash
❯ sbctl status
Installed:	✓ sbctl is installed
Owner GUID:	3b1a53a7-ae4f-47bb-92af-67c2d769cceb
Setup Mode:	✓  Enabled
Secure Boot:	✓ Disabled
Vendor Keys:	microsoft
```

Indicando que eu tenho uma chave, pois eu já tinha criado, ele mostra também que eu tenho o secure boot em modo setup para configuração, esta mostrando o *Vendor Keys: microsoft* pois eu tenho uma instalação valida do windows, a microsoft assina todas suas chaves então é por isso que nunca precisamos assinar, já no linux a coisa é um pouco diferente, vamos assinar isso manualmente.

Depois de rodar *sbctl create-keys* ele cria uma key para assinar nossas imagens efi, vamos registrar as chaves incluindo as da Microsoft: *sbctl enroll-keys -m*.

Nesse próximo passo você faz a verificação de quais keys você já possui:

```bash 
sudo sbctl verify
```

Ele retorna todas as chaves que você tem, assinadas e não assinadas.

como eu uso do GRUB com Secure Boot e chaves personalizadas requer um passo extra muito importante: é necessário reinstalar o GRUB com parâmetros específicos para evitar que ele bloqueie o arranque por não encontrar o "Shim" (um intermediário de assinatura que não estamos a usar).

Execute o comando de instalação do GRUB novamente (ajuste o `--efi-directory` se o seu não for `/boot` ou `/boot/efi`):

```bash
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB --modules="tpm" --disable-shim-lock
```

Após isso, regenere o ficheiro de configuração:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Agora assinamos o binário do GRUB (que acabámos de gerar) e o Kernel.

Você **não** precisa (e nem deve) assinar os arquivos do Windows (`bootmgfw.efi`), pois eles já são assinados pela Microsoft e, graças ao passo anterior, sua BIOS já confia neles. Assine apenas o Kernel do Arch e o seu Bootloader:

```bash
sudo sbctl sign -s /boot/vmlinuz-linux
sudo sbctl sign -s /boot/EFI/GRUB/grubx64.efi
```

%% (Se usar kernel LTS ou Zen, assine também, por exemplo: `vmlinuz-linux-lts`) %%

Vamos garantir que nada ficou para traz, rodando o comando *sudo sbctl verify*. Ele vai mostrar todos assinados.

Agora você só precisa reiniciar e ativar o secure boot, que já vai estar funcionando.

No próximo passo eu vou trazer um Archlinux criptografado.