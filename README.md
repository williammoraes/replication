[POC] Pooling em aplicação Rails
============================

Este é um simples exemplo de projeto trabalhando com replicação read/write. Com o objetivo de tornar simples o entendimento,  a aplicação conta apenas com um pequeno cadastro de contatos.


O bravo 'Oracle'
-------------------

  Para simular dois bancos de dados distintos, subi duas imagens do container **wnameless/oracle-xe-11g** no Docker. Por convenção subi as portas com um intervalo de 100.

    docker run -d -p 49160:22 -p 49161:1521 wnameless/oracle-xe-11g

    docker run -d -p 49260:22 -p 49261:1521 wnameless/oracle-xe-11g

  Ambas as conexões do oracle estão configuradas no  arquivo "tnsnames.ora". Dessa forma você não precisa passar a configuração completa da conexão na aplicação, apenas o tnsname que deseja.
> **Nota:**
> - A conexão por TNS depende da variável de ambiente que aponta para o diretório onde o arquivo se encontra: 
>  export TNS_ADMIN=/opt/oracle/instantclient_11_2/network/admin
> - Antes de conectar com o usuário abaixo você precisa criar o usuário e senha desejado.

       SHARED_LIST_DEV =
      (DESCRIPTION =
       (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.99.100)(PORT = 49161))
       (LOAD_BALANCE = yes)
       (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = XE)
        (FAILOVER_MODE =
         (TYPE = SELECT)
         (METHOD = BASIC)
         (RETRIES = 180)
         (DELAY = 5)
       )
     )
    )
    
    SHARED_LIST_DEV_2 =
      (DESCRIPTION =
       (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.99.100)(PORT = 49261))
       (LOAD_BALANCE = yes)
       (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = XE)
        (FAILOVER_MODE =
         (TYPE = SELECT)
         (METHOD = BASIC)
         (RETRIES = 180)
         (DELAY = 5)
       )
     )
    )

A conexão com o banco pode ser testada pelo client do oracle **sqlplus** com o comando:

    sqlplus $USERNAME/$PASSWD@$TNSNAME
    
    sqlplus SHARED_LIST_DEV/SHARED_LIST_DEV@SHARED_LIST_DEV


O destemido 'Octopus'
-----------

O pooling de conexões esta implementado em apenas um ambiente **"development"**, e é gerenciado pela gem **"[ar-octopus](https://github.com/tchandy/octopus)"**.
O octopus permite você configurar **replication** e **sharding** de maneira bem simples. 

A base de configuração do octopus é o arquivo **shards.yml**, que deve ficar na pasta **RAILS_APP/config** do projeto. Por padrão o **master/writer** será a conexão no 'database.yml'.

` shards.yml`

    octopus:
      replicated: true
      environments:
        - development
      development:
        slave1:
          adapter: oracle_enhanced
          database: SHARED_LIST_DEV_2
          username: SHARED_LIST_DEV
          password: SHARED_LIST_DEV

Ok, agora que você viu o arquivo deve estar pensando "O que mais eu posso fazer?!". A resposta é, muita coisa. Como o ActiveRecord e o Octopus são bem flexíveis, então vamos as possíveis perguntas:

####Posso configurar bancos diferentes nos slaves?
> Sim, basta ter o adapter instalado e passar as configurações nos shards(slaves) desejados.

#### Posso ter outros slaves e outros nomes configurados por ambientes? 
> Por que não? Vamos fazer:
> 

    octopus:
      replicated: true
      environments:
        - development
        - production
      development:
        so_leitura:
          adapter: oracle_enhanced
          database: SHARED_LIST_DEV_2
          username: SHARED_LIST_DEV
          password: ( ͡° ͜ʖ ͡°)
      production:
         brazil:
          adapter: mysql
          database: replication_prod
          username: root
          password: ( ͡° ͜ʖ ͡°)
         usa:
          adapter: mysql
          database: replication_prod
          username: root
          password: ( ͡° ͜ʖ ͡°)
        
#### Humm... interessante, mas eu não quero replicar tudo, apenas alguns models?!
> Tudo bem, o octopus tem um propriedade chamada **fully_replicated** que por default é **true**, mas podemos passar **false** e apartir de então só os models com o  método **replicated_model()** serão replicados.
> Exemplos:
> 

    
    #contact.rb
    #Esta classe é replicada, escritas na master and leitura na slave.
    class Contact < ActiveRecord::Base
      replicated_model()
    end

    #shards.yml
        octopus:
          replicated: true
          fully_replicated: false
          environments:
            - development
          development:
            slave1:
              adapter: oracle_enhanced
              database: SHARED_LIST_DEV_2
              username: SHARED_LIST_DEV
              password: SHARED_LIST_DEV

#### Tudo bem, eu quero replicar, mas eu tenho grupos de slaves por região, o que eu faço?
> Sem problemas, você pode agrupar os servidores ;)
>

     octopus:
      replicated: true
      environments:
        - development
      development:
        slave_group_a:
          slave1:
            # ...
          slave2:
            # ...
        slave_group_b:
          slave3:
            # ...
          slave4:
            # ...

Migrations
-------------
Por padrão as migrations são rodadas apenas na master, caso você deseje que elas rodem nos slaves/shards, basta especificar quem você quer utilizar. Exemplo:

    class CreateContactsUsingBlockAndUsing < ActiveRecord::Migration
      using(:brazil, :argentina)
    
      def self.up
        using(:usa) do
          Contact.create!(:name => "Tio San")
        end
    
        Contact.create!(:name => "O Pelé é melhor")
      end
    
      def self.down
        Contact.delete_all()
      end
    end 

##Using o Octopus
Com a replicação configurada você não precisa fazer mais nada,
todas as queries de leitura trabalharam com os slaves e as escritas com o master.
Porém, você também pode especificar manualmente de onde quer que os dados venham, exemplo:

    @brazil_contact = Contact.using(:brazil).find_by_name("Zé")

E se eu quiser trabalhar com grupos:

    Contact.using(slave_group: :slave_group_a).all
  Pronto, todos os selects estarão rodando nos slaves do grupo especificado. 

#That's all =)