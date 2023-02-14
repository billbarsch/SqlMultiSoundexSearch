# Busca de palavras em banco de dados MySQL com letras e sílabas erradas

Este artigo descreve como buscar palavras dentro de um banco de dados MySQL com letras e sílabas erradas para encontrar o registro correto.

## 1. Função Soundex
A função Soundex do MySQL é útil, mas não funciona muito bem para o português brasileiro. Para contornar isso, foi criada a função soundex_ptbr, que é adaptada ao nosso idioma.

```SQL
DELIMITER $$
CREATE FUNCTION `soundex_ptbr`(`str` varchar(255)) RETURNS varchar(10) CHARSET utf8
BEGIN
  DECLARE len INT DEFAULT LENGTH(str);
  DECLARE i INT DEFAULT 1;
  DECLARE j INT DEFAULT 1;
  DECLARE first_char CHAR(1);
  DECLARE new_str VARCHAR(255) DEFAULT '';
  DECLARE last_code CHAR(1);
  DECLARE this_code CHAR(1);
  DECLARE soundex_code VARCHAR(10) DEFAULT '';

-- Substitui caracteres acentuados e especiais
  SET new_str = remove_accents(UPPER(str));

  -- Primeira letra da palavra
  SET soundex_code = LEFT(new_str, 1);
  SET last_code = soundex_code;

  WHILE (i < len AND j < 4) DO
    SET i = i + 1;
    SET this_code = NULL;

    -- Ignora acentos e cedilhas
    IF SUBSTRING(new_str, i, 1) IN ('A', 'E', 'I', 'O', 'U', 'H', 'W', 'Y') THEN
      SET this_code = '0';
    ELSEIF SUBSTRING(new_str, i, 1) IN ('B', 'P') THEN
      SET this_code = '1';
    ELSEIF SUBSTRING(new_str, i, 1) IN ('C', 'G', 'J', 'K', 'Q', 'S', 'X', 'Z') THEN
      SET this_code = '2';
    ELSEIF SUBSTRING(new_str, i, 1) IN ('D', 'T') THEN
      SET this_code = '3';
    ELSEIF SUBSTRING(new_str, i, 1) IN ('F', 'V') THEN
      SET this_code = '4';
    ELSEIF SUBSTRING(new_str, i, 1) = 'L' THEN
      SET this_code = '5';
    ELSEIF SUBSTRING(new_str, i, 1) IN ('M', 'N') THEN
      SET this_code = '6';
    ELSEIF SUBSTRING(new_str, i, 1) = 'R' THEN
      SET this_code = '7';
    ELSEIF SUBSTRING(new_str, i, 1) = 'Ç' THEN
      SET this_code = '8';
    END IF;

    IF (this_code IS NOT NULL AND this_code != last_code) THEN
      SET soundex_code = CONCAT(soundex_code, this_code);
      SET j = j + 1;
    END IF;

    SET last_code = this_code;
  END WHILE;

  -- Completar o soundex com zeros caso necessário
  SET soundex_code = CONCAT(soundex_code, REPEAT('0', 4-j));

  RETURN soundex_code;
END
$$
DELIMITER ;
```

## 2. Remoção de acentos
A função soundex_ptbr estava criando códigos fonéticos com perfeição, mas ainda estava deixando passar letras acentuadas. Para resolver esse problema, foi criada a função remove_accents, que filtra as letras acentuadas e as substitui por letras sem acento. Essa função foi aplicada na função soundex_ptbr.

```SQL
DELIMITER $$
CREATE FUNCTION `remove_accents`(str VARCHAR(255)) RETURNS varchar(255) CHARSET utf8
BEGIN
  DECLARE len INT DEFAULT LENGTH(str);
  DECLARE i INT DEFAULT 1;
  DECLARE c CHAR(1);
  DECLARE new_str VARCHAR(255) DEFAULT '';

  WHILE (i <= len) DO
    SET c = SUBSTRING(str, i, 1);
    IF (c IN ('À', 'Á', 'Â', 'Ã', 'Ä')) THEN
      SET new_str = CONCAT(new_str, 'A');
    ELSEIF (c IN ('à', 'á', 'â', 'ã', 'ä')) THEN
      SET new_str = CONCAT(new_str, 'a');
    ELSEIF (c = 'Ç') THEN
      SET new_str = CONCAT(new_str, 'C');
    ELSEIF (c = 'ç') THEN
      SET new_str = CONCAT(new_str, 'c');
    ELSEIF (c IN ('È', 'É', 'Ê', 'Ë')) THEN
      SET new_str = CONCAT(new_str, 'E');
    ELSEIF (c IN ('è', 'é', 'ê', 'ë')) THEN
      SET new_str = CONCAT(new_str, 'e');
    ELSEIF (c IN ('Ì', 'Í', 'Î', 'Ï')) THEN
      SET new_str = CONCAT(new_str, 'I');
    ELSEIF (c IN ('ì', 'í', 'î', 'ï')) THEN
      SET new_str = CONCAT(new_str, 'i');
    ELSEIF (c IN ('Ò', 'Ó', 'Ô', 'Õ', 'Ö')) THEN
      SET new_str = CONCAT(new_str, 'O');
    ELSEIF (c IN ('ò', 'ó', 'ô', 'õ', 'ö')) THEN
      SET new_str = CONCAT(new_str, 'o');
    ELSEIF (c IN ('Ù', 'Ú', 'Û', 'Ü')) THEN
      SET new_str = CONCAT(new_str, 'U');
    ELSEIF (c IN ('ù', 'ú', 'û', 'ü')) THEN
      SET new_str = CONCAT(new_str, 'u');
    ELSEIF (c = 'Ñ') THEN
      SET new_str = CONCAT(new_str, 'N');
    ELSEIF (c = 'ñ') THEN
      SET new_str = CONCAT(new_str, 'n');
    ELSE
      SET new_str = CONCAT(new_str, c);
    END IF;

    SET i = i + 1;
  END WHILE;

  RETURN new_str;
END
$$
DELIMITER ;
```

## 3. Aplicação em strings com várias palavras
A função soundex_ptbr converte uma palavra para um código fonético, mas para uma string com várias palavras não estava funcionando como deveria. Para solucionar esse problema, foi criada a função multisoundex, que aplica a soundex_ptbr em cada palavra de uma string e converte cada palavra separadamente.

```SQL
DELIMITER $$
CREATE FUNCTION `multisoundex`(`str` varchar(255)) RETURNS varchar(255) CHARSET utf8
BEGIN
  DECLARE result VARCHAR(255) DEFAULT '';
  DECLARE word VARCHAR(255) DEFAULT '';
  DECLARE i INT DEFAULT 1;
  DECLARE len INT DEFAULT LENGTH(str);

  WHILE (i <= len) DO
    -- Enquanto o caractere atual for um espaço ou um traço, avance para o próximo caractere
    WHILE (i <= len AND (SUBSTRING(str, i, 1) = ' ' OR SUBSTRING(str, i, 1) = '-')) DO
      SET i = i + 1;
    END WHILE;

    -- Extrai a palavra atual
    SET word = '';
    WHILE (i <= len AND SUBSTRING(str, i, 1) != ' ' AND SUBSTRING(str, i, 1) != '-') DO
      SET word = CONCAT(word, SUBSTRING(str, i, 1));
      SET i = i + 1;
    END WHILE;

    -- Se a palavra atual não for vazia, aplica a função soundex_ptbr a ela e concatena com o resultado
    IF (word != '') THEN
      SET result = CONCAT(result, soundex_ptbr(word));
    END IF;

    -- Adiciona um espaço ao resultado se ainda houver palavras para processar
    IF (i <= len) THEN
      SET result = CONCAT(result, ' ');
    END IF;
  END WHILE;

  RETURN result;
END
$$
DELIMITER ;
```

## 4. Verificação de existência de palavra em uma string
Tendo a função multisoundex, agora é possível verificar se uma string menor de busca existe dentro de uma string maior (banco de dados) e, assim, saber se a palavra buscada existe ou não. Isso foi feito criando-se uma nova função chamada hassoundex.

```SQL
DELIMITER $$
CREATE FUNCTION `hassoundex`(`str2` varchar(400), `str1` varchar(400)) RETURNS tinyint(1)
BEGIN
  DECLARE soundex_str1 VARCHAR(400);
  DECLARE soundex_str2 VARCHAR(400);

  SET soundex_str1 = REPLACE(multisoundex(str1), ' ', '%');
  SET soundex_str2 = multisoundex(str2);

  IF soundex_str2 LIKE CONCAT('%', soundex_str1, '%') THEN
    RETURN TRUE;
  ELSE
    RETURN FALSE;
  END IF;
END
$$
DELIMITER ;
```

## 5. Comparação de similaridade entre strings
Usando a função hassoundex, os resultados eram perfeitos, mas o melhor resultado nem sempre aparecia em primeiro lugar. Para resolver esse problema, foi criada a função levenshtein, que compara duas strings e diz em uma escala numérica o quão parecidas elas são. Essa função foi usada no Order By da busca.

```SQL
DELIMITER $$
CREATE FUNCTION `LEVENSHTEIN`(s1 VARCHAR(255), s2 VARCHAR(255)) RETURNS int(11)
    DETERMINISTIC
BEGIN
    DECLARE s1_len, s2_len, i, j, c, c_temp, cost INT;
    DECLARE s1_char CHAR;
    DECLARE cv0, cv1 VARBINARY(256);
    SET s1_len = CHAR_LENGTH(s1), s2_len = CHAR_LENGTH(s2), cv1 = 0x00, j = 1, i = 1, c = 0;
    IF s1 = s2 THEN
        RETURN 0;
    ELSEIF s1_len = 0 THEN
        RETURN s2_len;
    ELSEIF s2_len = 0 THEN
        RETURN s1_len;
    ELSE
        WHILE j <= s2_len DO
            SET cv1 = CONCAT(cv1, UNHEX(HEX(j))), j = j + 1;
        END WHILE;
        WHILE i <= s1_len DO
            SET s1_char = SUBSTRING(s1, i, 1), c = i, cv0 = UNHEX(HEX(i)), j = 1;
            WHILE j <= s2_len DO
                SET c = c + 1;
                IF s1_char = SUBSTRING(s2, j, 1) THEN SET cost = 0; ELSE SET cost = 1; END IF;
                SET c_temp = CONV(HEX(SUBSTRING(cv1, j, 1)), 16, 10) + cost;
                IF c > c_temp THEN SET c = c_temp; END IF;
                SET c_temp = CONV(HEX(SUBSTRING(cv1, j+1, 1)), 16, 10) + 1;
                IF c > c_temp THEN SET c = c_temp; END IF;
                SET cv0 = CONCAT(cv0, UNHEX(HEX(c))), j = j + 1;
            END WHILE;
            SET cv1 = cv0, i = i + 1;
        END WHILE;
    END IF;
    RETURN c;
END
$$
DELIMITER ;
```

# Conclusão

O uso de todas essas funções em conjunto permite que sejam feitas buscas por palavras com muitos erros e diferenças de grafias fortes e, mesmo assim, encontrar o registro perfeito que mais se encaixa no desejo do usuário. Exemplos de busca incluem "Camisas magas longass" que encontra o produto "Camisa Manga Longa" e "Evando Gomes Lindoberg" que acha o cliente "Evandson Gomes Lindhemberg".

## Exemplos

Em um banco de dados com os seguintes registros:

``` 
razaoSocial
EMILLY EVILLY VERAS CORDEIRO
EMILLY NEVES ALVES MOURA
Emily Evely Santos Lima
EMILY JULIA EVARISTO DE SOUSA
```

A busca abaixo:

```SQL
SELECT razaoSocial 
FROM basicoParticipante 
WHERE hassoundex(razaoSocial, 'emeli eveli')
ORDER BY LEVENSHTEIN(razaoSocial, 'emeli eveli')
```

Retorna:

```
razaoSocial
Emily Evely Santos Lima
EMILLY EVILLY VERAS CORDEIRO
```

Pode-se escrever com vários erros:
```SQL
SELECT razaoSocial 
FROM basicoParticipante 
WHERE hassoundex(razaoSocial, 'emile evelli santo')
ORDER BY LEVENSHTEIN(razaoSocial, 'emile evelli santo')
```

Retorna:

```
razaoSocial
Emily Evely Santos Lima
```


