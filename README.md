# Ambiente de Desenvolvimento WordPress para ABPC

Este README fornece instruções para configurar um ambiente de desenvolvimento local para um site WordPress da **Associação Brasiliense de Peritos em Criminalística (ABPC)** usando Docker e Docker Compose no Ubuntu, com credenciais armazenadas de forma segura em um arquivo `.env`. Inclui controle de versão com Git e GitHub, backup e restauração do ambiente, e implantação do site final no serviço de hospedagem.

## 1. Configurando o Ambiente de Desenvolvimento Local

Este setup assume que você está usando Ubuntu 22.04 LTS ou posterior. Usaremos Docker para containerizar o WordPress (Apache/PHP) e um banco de dados MySQL, com a palavra-chave "abpc" integrada à configuração para refletir a identidade da ABPC.

### Pré-requisitos
- Ubuntu 22.04 LTS ou posterior instalado na sua máquina.
- Familiaridade básica com o terminal.

### Instruções Passo a Passo

1. **Atualizar os Pacotes do Sistema**:
   Abra um terminal e execute:
   ```
   sudo apt update && sudo apt upgrade -y
   ```

2. **Instalar o Docker**:
   Instale o Docker Engine e ferramentas relacionadas:
   ```
   sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   sudo apt update
   sudo apt install docker-ce docker-ce-cli containerd.io -y
   ```
   Adicione seu usuário ao grupo Docker:
   ```
   sudo usermod -aG docker $USER
   ```
   Faça logout e login novamente. Verifique a instalação do Docker:
   ```
   docker --version
   ```

3. **Instalar o Docker Compose**:
   Instale o Docker Compose:
   ```
   sudo apt install docker-compose -y
   ```
   Verifique a instalação:
   ```
   docker compose version
   ```

4. **Criar um Diretório para o Projeto**:
   Crie um diretório para o projeto ABPC WordPress:
   ```
   mkdir ~/abpc-wordpress && cd ~/abpc-wordpress
   ```

5. **Criar o Arquivo .env**:
   Crie um arquivo `.env` para armazenar variáveis de ambiente sensíveis e o nome do projeto:
   ```
   nano .env
   ```
   Adicione o seguinte conteúdo (substitua as senhas por valores seguros):
   ```
   COMPOSE_PROJECT_NAME=abpc
   MYSQL_ROOT_PASSWORD=secure_root_password_123
   MYSQL_DATABASE=wordpress
   MYSQL_USER=wordpress_user
   MYSQL_PASSWORD=secure_user_password_456
   WORDPRESS_DB_HOST=db:3306
   WORDPRESS_DB_USER=wordpress_user
   WORDPRESS_DB_PASSWORD=secure_user_password_456
   WORDPRESS_DB_NAME=wordpress
   ```
   Garanta que o arquivo `.env` não seja incluído no controle de versão (veja a seção Git).

6. **Criar o Arquivo Docker Compose**:
   Crie um arquivo chamado `docker-compose.yml` com o seguinte conteúdo, usando uma rede interna segura e nomes de contêineres e volumes com a marca ABPC:
   ```yaml
	services:
	  db:
	    image: mysql:8.0.39
	    container_name: abpc_db
	    volumes:
	      - abpc_db_data:/var/lib/mysql
	    restart: always
	    env_file:
	      - .env
	    environment:
	      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
	      MYSQL_DATABASE: ${MYSQL_DATABASE}
	      MYSQL_USER: ${MYSQL_USER}
	      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
	    networks:
	      - wordpress_network
	    healthcheck:
	      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
	      interval: 10s
	      timeout: 5s
	      retries: 5

	  wordpress:
	    depends_on:
	      db:
	        condition: service_healthy
	    image: wordpress:latest
	    container_name: abpc_web
	    volumes:
	      - abpc_wordpress_data:/var/www/html
	    ports:
	      - "8000:80"
	    restart: always
	    env_file:
	      - .env
	    environment:
	      WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
	      WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
	      WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
	      WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
	    networks:
	      - wordpress_network

	networks:
	  wordpress_network:
	    driver: bridge
	    name: abpc_network

	volumes:
	  abpc_db_data:
	    name: abpc_db_data
	  abpc_wordpress_data:
	    name: abpc_wordpress_data
   ```

7. **Iniciar os Contêineres**:
   Execute:
   ```
   docker compose up -d
   ```
   Acesse o WordPress em `http://localhost:8000` e complete o assistente de configuração, definindo o título do site como "ABPC" ou "Associação Brasiliense de Peritos em Criminalística".

8. **Parar os Contêineres** (Quando Necessário):
   ```
   docker compose down
   ```

## 2. Controle de Versão com Git e GitHub

Use o Git para rastrear alterações nos arquivos do projeto (por exemplo, temas personalizados como `abpc-theme`, plugins ou `docker-compose.yml`).

### Instruções Passo a Passo

1. **Instalar o Git** (se ainda não instalado):
   ```
   sudo apt install git -y
   ```

2. **Inicializar o Git no Diretório do Projeto**:
   Navegue até o diretório do projeto:
   ```
   cd ~/abpc-wordpress
   git init
   ```

3. **Criar um Arquivo .gitignore**:
   Crie um arquivo `.gitignore` com:
   ```
   # Ignorar dados dos volumes do Docker
   abpc_db_data/
   abpc_wordpress_data/

   # Ignorar arquivo de ambiente sensível
   .env

   # Ignorar arquivos centrais do WordPress (se montados localmente)
   wp-admin/
   wp-includes/
   wp-content/uploads/  # Rastrear temas/plugins personalizados, ex.: wp-content/themes/abpc-theme
   ```

4. **Adicionar e Fazer Commit dos Arquivos**:
   ```
   git add .
   git commit -m "Commit inicial: Configuração segura do Docker para o WordPress da ABPC"
   ```

5. **Configurar o Repositório no GitHub**:
   - Crie um novo repositório no GitHub chamado `abpc-wordpress`.
   - Copie a URL do repositório (ex.: `https://github.com/AdmilsonGoncalves/abpc-wordpress.git`).

6. **Configurar Autenticação no GitHub**:
   - **Nota**: O GitHub não suporta autenticação por senha para operações Git. Use um **personal access token** (PAT) ou chave SSH.
   - **Opção 1: Personal Access Token (HTTPS)**:
     - No GitHub, vá para **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)** > **Generate new token**.
     - Selecione o escopo `repo` (e outros, se necessário, como `workflow` para GitHub Actions) e gere o token. Copie-o (ex.: `ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`).
     - **Vincular o Token**:
       - Configure o Git para armazenar credenciais localmente, permitindo que o token seja usado automaticamente em operações futuras:
         ```
         git config --global credential.helper store
         ```
       - **Acessar o Repositório com o Token**:
         - Ao executar comandos como `git push`, `git pull` ou `git clone`, use a URL HTTPS do repositório (ex.: `https://github.com/AdmilsonGoncalves/abpc-wordpress.git`). Quando solicitado, insira seu nome de usuário (ex.: `AdmilsonGoncalves`) e o token como senha. Exemplo de comando para clonar:
           ```
           git clone https://github.com/AdmilsonGoncalves/abpc-wordpress.git
           ```
         - Após o primeiro uso, o token será armazenado (se `credential.helper store` estiver configurado) e não será necessário inseri-lo novamente.
     - Alternativamente, para evitar prompts de autenticação, você pode incorporar o token diretamente na URL do repositório (não recomendado para scripts públicos, pois expõe o token):
       ```
       git clone https://AdmilsonGoncalves:ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx@github.com/AdmilsonGoncalves/abpc-wordpress.git
       ```
   - **Opção 2: SSH**:
     - Gere uma chave SSH:
       ```
       ssh-keygen -t ed25519 -C "seu_email@example.com"
       ```
     - Adicione a chave pública ao GitHub (**Settings** > **SSH and GPG keys**).
     - Atualize o remote para SSH:
       ```
       git remote set-url origin git@github.com:AdmilsonGoncalves/abpc-wordpress.git
       ```

7. **Vincular e Enviar para o GitHub**:
   ```
   git remote add origin https://github.com/AdmilsonGoncalves/abpc-wordpress.git
   git branch -M main
   git push -u origin main
   ```
   Se usar HTTPS, insira seu nome de usuário e o token quando solicitado. Se usar SSH, o push será autenticado automaticamente.


## 3. ‘Backup’ e Restauração do Ambiente

Os ‘backups’ incluem arquivos do WordPress, o banco de dados MySQL e o arquivo `.env`.

### Instruções de ‘Backup’

1. **Parar os Contêineres** (Opcional, mas recomendado):
   ```
   docker compose down
   ```

2. **Fazer Backup dos Arquivos do WordPress**:
   Arquive o volume `abpc_wordpress_data`:
   ```
   docker run --rm --volumes-from abpc_web -v $(pwd):/backup ubuntu tar cvf /backup/abpc_wordpress_files.tar /var/www/html
   ```

3. **Fazer Backup do Banco de Dados**:
   Exporte o SQL:
   ```
   docker compose exec abpc_db mysqldump -u ${MYSQL_USER} -p${MYSQL_PASSWORD} ${MYSQL_DATABASE} > abpc_db_backup.sql
   ```

4. **Arquivar Tudo**:
   Inclua configurações e ‘backups’:
   ```
   tar -czvf abpc_full_backup.tar.gz docker-compose.yml .gitignore .env abpc_db_backup.sql abpc_wordpress_files.tar
   ```
   Transfira `abpc_full_backup.tar.gz` para outro computador.

### Instruções de Restauração em Outro Computador

1. **Configurar a Nova Máquina**:
   Siga a Seção 1 (passos 1–3) para instalar Docker e Docker Compose.

2. **Extrair o Backup**:
   ```
   tar -xzvf abpc_full_backup.tar.gz
   ```

3. **Restaurar Arquivos do WordPress**:
   Inicie os contêineres e restaure:
   ```
   docker compose up -d
   docker run --rm --volumes-from abpc_web -v $(pwd):/backup ubuntu tar xvf /backup/abpc_wordpress_files.tar -C /
   ```

4. **Restaurar o Banco de Dados**:
   ```
   cat abpc_db_backup.sql | docker compose exec -T abpc_db mysql -u ${MYSQL_USER} -p${MYSQL_PASSWORD} ${MYSQL_DATABASE}
   ```

5. **Reiniciar os Contêineres**:
   ```
   docker compose restart
   ```
   Acesse em `http://localhost:8000`.

## 4. Conectando e Baixando o Repositório do GitHub

Esta seção descreve como clonar o repositório do projeto `abpc-wordpress` do GitHub para sua máquina local e configurar o ambiente para desenvolvimento, garantindo que você possa trabalhar com os arquivos do projeto ABPC.

### Pré-requisitos
- Git instalado (veja Seção 2, passo 1).
- Acesso ao repositório GitHub `abpc-wordpress` (ex.: `https://github.com/AdmilsonGoncalves/abpc-wordpress.git`).
- Autenticação configurada (HTTPS com Personal Access Token ou SSH, conforme descrito na Seção 2, passo 6).

### Instruções Passo a Passo

1. **Clonar o Repositório**:
   No terminal, navegue até o diretório onde deseja armazenar o projeto e clone o repositório:
	```
	cd ~
	git clone https://github.com/AdmilsonGoncalves/abpc-wordpress.git
	```

## 5. Implantando o Site no Hostinger

### Instruções Passo a Passo

1. **Preparar o Site Local para Exportação**:
   - Instale o plugin "All-in-One WP Migration" via painel do WordPress (`http://localhost:8000/wp-admin`).
   - Exporte para um arquivo `.wpress` (inclui arquivos e banco de dados).

   Alternativamente, exportação manual:
   - Exporte o banco: Use `abpc_db_backup.sql` da Seção 3.
   - Compacte os arquivos do WordPress de `/var/www/html`.

2. **Configurar o WordPress no Hostinger**:
   - Faça login no painel do Hostinger.
   - Vá para Websites > Adicionar Website > Selecionar WordPress.
   - Conclua a configuração com um site temporário.

3. **Importar para o Hostinger**:
   - Instale o "All-in-One WP Migration" no site WordPress do Hostinger.
   - Importe o arquivo `.wpress`.
   - Alternativamente, faça upload dos arquivos compactados para `/public_html` via Gerenciador de Arquivos ou FTP, crie um banco MySQL, importe `abpc_db_backup.sql` via phpMyAdmin, atualize `wp-config.php` com as credenciais do Hostinger e substitua `localhost:8000` pelo domínio usando um plugin como Better Search Replace.

4. **Passos Pós-Implantação**:
   - Atualize os permalinks para incluir `/abpc/` se desejar (ex.: `/abpc/%postname%/`).
   - Defina o título do site como "ABPC" ou "Associação Brasiliense de Peritos em Criminalística".
   - Teste o site, aponte o DNS do domínio para os nameservers do Hostinger e ative o AutoSSL.

## 6. Limpando e Reaplicando o Ambiente Docker

Caso sejam feitas alterações no arquivo `docker-compose.yml`, no arquivo `.env`, ou em outros componentes do ambiente (como imagens Docker ou configurações), pode ser necessário limpar o ambiente existente e recriá-lo para aplicar as modificações corretamente. Esta seção descreve como limpar o ambiente Docker do projeto ABPC WordPress e reiniciá-lo.

### Pré-requisitos
- Docker e Docker Compose instalados (veja Seção 1, passos 2 e 3).
- O projeto já configurado no diretório `~/abpc-wordpress` (veja Seção 5).
- **Aviso**: A limpeza removerá os contêineres, redes e volumes. Faça backup dos dados importantes antes de prosseguir (veja Seção 3).

### Instruções Passo a Passo

1. **Navegar até o Diretório do Projeto**:
   Acesse o diretório do projeto:
   ```
   cd ~/abpc-wordpress
   ```

2. **Parar e Remover os Contêineres**:
   Pare e remova os contêineres criados pelo `docker-compose.yml`:
   ```
   docker compose down
   ```
   Este comando para os contêineres `abpc_web` e `abpc_db` e remove a rede `abpc_network`.

3. **Remover os Volumes (Opcional)**:
   Se as alterações exigirem uma reinicialização completa (por exemplo, mudanças nas configurações do banco de dados ou arquivos do WordPress), remova os volumes associados:
   ```
   docker volume rm abpc_db_data abpc_wordpress_data
   ```
   **Nota**: Isso apagará o banco de dados e os arquivos do WordPress. Certifique-se de ter um backup (veja Seção 3) se precisar restaurar os dados posteriormente.

4. **Remover Imagens Docker (Opcional)**:
   Se você alterou a versão das imagens (ex.: `mysql:8.0.39` ou `wordpress:latest`) no `docker-compose.yml`, pode ser necessário remover as imagens antigas:
   ```
   docker image rm mysql:8.0.39 wordpress:latest
   ```
   Isso força o Docker a baixar as versões mais recentes das imagens ao reiniciar.

5. **Aplicar as Alterações**:
   Após realizar as alterações necessárias no `docker-compose.yml` ou `.env`, recrie e inicie o ambiente:
   ```
   docker compose up -d --build
   ```
   - A opção `--build` garante que quaisquer mudanças na configuração sejam aplicadas.
   - O Docker baixará as imagens necessárias (se removidas) e recriará os contêineres, volumes e redes.

6. **Verificar o Ambiente**:
   Confirme que os contêineres estão funcionando:
   ```
   docker compose ps
   ```
   Acesse o WordPress em `http://localhost:8000`. Se os volumes foram removidos, complete o assistente de configuração do WordPress novamente, definindo o título como "ABPC" ou "Associação Brasiliense de Peritos em Criminalística".

7. **Restaurar Dados (Se Necessário)**:
   Se você fez backup anteriormente (Seção 3) e removeu os volumes, restaure os dados conforme descrito na Seção 3, passos de restauração.

### Notas
- **Cuidado com a remoção de volumes**: A remoção dos volumes `abpc_db_data` e `abpc_wordpress_data` é irreversível sem backup. Sempre execute a Seção 3 (Instruções de Backup) antes de limpar os volumes.
- **Alterações leves**: Se as mudanças no `docker-compose.yml` forem pequenas (ex.: alteração de portas ou variáveis de ambiente), apenas o comando `docker compose down` seguido de `docker compose up -d` pode ser suficiente, sem remover volumes ou imagens.
- **Manutenção regular**: Para evitar conflitos, limpe imagens e contêineres não utilizados periodicamente com:
  ```
  docker system prune -a
  docker volume prune
  ```
  Use esses comandos com cuidado, pois eles afetam todos os projetos Docker na máquina.
