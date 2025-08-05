# Manual de Deploy na VPS com Portainer e Traefik

Este guia descreve como implantar o Blinko em uma VPS que já possui **Portainer** e **Traefik** configurados. A implantação será feita por meio de uma *stack* do Portainer utilizando o `docker-compose` de produção.

## Pré-requisitos

- Docker e Docker Compose instalados.
- A VPS deve possuir Portainer e Traefik em execução.
- Rede do Traefik (ex.: `traefik`) já criada e configurada como externa.
- Domínio apontando para a VPS.

## Passo a passo

1. **Preparar variáveis de ambiente**  
   Defina os valores desejados para `NEXTAUTH_SECRET`, `DATABASE_URL` e outras variáveis necessárias. Opcionalmente você pode utilizar um arquivo `.env` no Portainer para manter os segredos.

2. **Criar uma nova Stack no Portainer**  
   - Acesse o painel do Portainer (`http://IP_DA_VPS:9000`).
   - No menu lateral, clique em **Stacks** e depois em **Add stack**.
   - Escolha um nome para a stack, por exemplo `blinko`.

3. **Inserir o arquivo de stack**  
   Cole o conteúdo YAML abaixo no editor do Portainer. Ajuste o domínio em `traefik.http.routers.blinko.rule` e as variáveis de ambiente conforme necessário.

   ```yaml
   version: "3.8"

   networks:
     blinko-network:
       driver: bridge
     traefik:
       external: true

   services:
     blinko-website:
       image: blinkospace/blinko:latest
       container_name: blinko-website
       environment:
         NODE_ENV: production
         NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
         DATABASE_URL: ${DATABASE_URL}
       depends_on:
         postgres:
           condition: service_healthy
       labels:
         - "traefik.enable=true"
         - "traefik.http.routers.blinko.rule=Host(`blinko.seudominio.com`)"
         - "traefik.http.routers.blinko.entrypoints=web,websecure"
         - "traefik.http.routers.blinko.tls.certresolver=letsencrypt"
         - "traefik.http.services.blinko.loadbalancer.server.port=1111"
       networks:
         - traefik
         - blinko-network
       restart: always
       healthcheck:
         test: ["CMD", "curl", "-f", "http://blinko-website:1111/"]
         interval: 30s
         timeout: 10s
         retries: 5
         start_period: 30s

     postgres:
       image: postgres:14
       container_name: blinko-postgres
       environment:
         POSTGRES_DB: postgres
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: mysecretpassword
       networks:
         - blinko-network
       restart: always
       healthcheck:
         test: ["CMD", "pg_isready", "-U", "postgres", "-d", "postgres"]
         interval: 5s
         timeout: 10s
         retries: 5
   ```

4. **Deploy**  
   Clique em **Deploy the stack** e aguarde o Portainer iniciar os containers.

5. **Verificar**  
   - Acesse os logs dos serviços para garantir que não há erros.
   - Abra o navegador e visite `https://blinko.seudominio.com` para confirmar que a aplicação está no ar.

## Atualizações

Para atualizar a versão do Blinko, acesse a stack no Portainer e clique em **Editor** > **Update the stack**. O Portainer fará o *pull* da nova imagem e reiniciará os serviços.

## Remoção

Para remover completamente a aplicação:

1. Exclua a stack no Portainer.
2. Remova volumes e redes se não forem mais necessários.

---

Com isso o Blinko estará disponível na sua VPS utilizando Portainer e Traefik.
