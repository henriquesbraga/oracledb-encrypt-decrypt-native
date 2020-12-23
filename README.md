<p align="center"><h2>Um exemplo de como usar funções de criptografia no Oracle Database com a package DBMS_CRYPTO.</h2></p>

Descobri um método de como usar funções de criptografia nativas no Oracle database (a partir do 10g). Apesar de hoje em dia as aplicações já enviarem para o banco dados como senha só com o hash, achei interessante esse método deixando essa responsabilidade para o banco de dados.

A versão 10g do Oracle Database trouxe a package DBMS_CRYPTO para criptografar e descriptografar dados. Usaremos dados do tipo texto (uma senha, para ser mais exato). Essa package está presente na versão 10g em diante.

Quero creditar o autor Mohammad Nazmul Huda. Ao ver seu artigo, me inspirei para fazer esse tutorial.

Primeiro, vamos criar a package e suas funções, logo em seguida criaremos uma tabela e testaremos a criptografia.


Começaremos criando uma package chamada encrypt_decrypt com as funções encrypt e decrypt. Faremos nossa conexão com o privilégio SYSDBA.

    --Cria a package
    CREATE OR REPLACE PACKAGE encrypt_decrypt
    AS
	    FUNCTION encrypt (p_plainText VARCHAR2) RETURN RAW DETERMINISTIC;
	    FUNCTION decrypt (p_encryptedText RAW) RETURN VARCHAR2 DETERMINISTIC;
    END;
    ============================================================================
		
    --Cria a package body
    CREATE OR REPLACE PACKAGE BODY encrypt_decrypt
    AS
	    encryption_type    PLS_INTEGER := DBMS_CRYPTO.ENCRYPT_DES
                                            + DBMS_CRYPTO.CHAIN_CBC
                                            + DBMS_CRYPTO.PAD_PKCS5;
	    encryption_key     RAW (32) := UTL_RAW.cast_to_raw('3NCR1PT3DK3Y');
	
    --A chave de encriptação deve ser maior que 8 bytes para o algoritimo DES (Data Encryption Standard).
    --Usaremos 3NCR1PT3DK3Y como chave.
	
	    FUNCTION encrypt (p_plainText VARCHAR2) RETURN RAW DETERMINISTIC
	    IS
		    encrypted_raw      RAW (2000);
	    BEGIN
		    encrypted_raw := DBMS_CRYPTO.ENCRYPT
	      (
	        src => UTL_RAW.CAST_TO_RAW (p_plainText),
	        typ => encryption_type,
	        key => encryption_key
	      );
	    RETURN encrypted_raw;
	    END encrypt;
	
	    FUNCTION decrypt (p_encryptedText RAW) RETURN VARCHAR2 DETERMINISTIC
	    IS
		    decrypted_raw      RAW (2000);
	    BEGIN
		    decrypted_raw := DBMS_CRYPTO.DECRYPT
	      (
	        src => p_encryptedText,
	        typ => encryption_type,
	        key => encryption_key
	      );
	    RETURN (UTL_RAW.CAST_TO_VARCHAR2 (decrypted_raw));
	    END decrypt;
    END;
    ============================================================================

    --Libera o acesso a package para o user
      GRANT EXECUTE ON encrypt_decrypt TO hr;

    --Cria o sinônimo para o package encrypt_decrypt
      create public synonym encrypt_decrypt for sys.encrypt_decrypt;
      
Agora, com o usuário hr criaremos a tabela, iremos inserir os dados e testar.

    --cria a tabela cliente
    CREATE TABLE cliente (
	    id              NUMBER,
	    name            VARCHAR2(30),
        username        VARCHAR2(30),
        password        VARCHAR2(200),
	    CONSTRAINT	customer_pk PRIMARY KEY(id)
    );

    --cria a sequencia
    CREATE SEQUENCE cliente_seq;

    --Insere os dados na tabela usando os métodos de criptografia
    INSERT INTO
	    cliente (id, name, username, password)
    VALUES
	    (cliente_seq.nextval, 'Cliente num. 1', 'cliente_1', encrypt_decrypt.encrypt('p4ssw0rd'));

    --Confere os dados
    SELECT * FROM cliente WHERE id = 1;
    Saída:  id = 1, name = 'Cliente num. 1', username = 'cliente_1', password = 'C5D62FE29ACAD280A551CF6A37116D3C'
    
    --Descriptografa a senha
    SELECT encrypt_decrypt.decrypt(password) AS password FROM cliente WHERE id = 1;
    Saída: password = p4ssw0rd
