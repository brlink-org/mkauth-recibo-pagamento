# Recibo Automático de Pagamento

Este projeto tem como objetivo enviar automaticamente um recibo de pagamento via WhatsApp quando o status de um pagamento é atualizado no banco de dados. Para isso, utilizamos uma *trigger* que insere os dados do pagamento em uma tabela auxiliar e um script PHP que processa e envia a mensagem.

Este projeto substitui o Addon https://github.com/brlink-org/mkauth-recibo-whatsapp que envia manualmente o recibo de pagamento.

## Passos para Configuração

### 1. Criar a Tabela Auxiliar

Primeiro, é necessário criar a tabela `brl_pago`, que armazenará os dados dos pagamentos que precisam ser processados e enviados via WhatsApp.

Execute o seguinte comando SQL para criar a tabela:

```sql
CREATE TABLE brl_pago (
    id INT(11) NOT NULL,
    login VARCHAR(64) NOT NULL,
    coletor VARCHAR(64),
    datavenc DATE,
    datapag DATETIME,
    valor DECIMAL(10, 2),
    valorpag DECIMAL(10, 2),
    formapag VARCHAR(32),
    PRIMARY KEY (id)
);
```

### 2. Criar a Trigger no Banco de Dados
Agora, crie uma trigger que será ativada após a atualização do status de pagamento na tabela `sis_lanc`. A trigger insere automaticamente os dados na tabela `brl_pago`.

Execute o seguinte comando SQL para criar a trigger:

```sql
CREATE TRIGGER tig_brl_pag
AFTER UPDATE ON sis_lanc
FOR EACH ROW
BEGIN
    -- Verifica se o status foi atualizado para "pago"
    IF NEW.status = 'pago' THEN
        -- Insere os dados na tabela auxiliar brl_pago
        INSERT INTO brl_pago (id, login, coletor, datavenc, datapag, valor, valorpag, formapag)
        VALUES (NEW.id, NEW.login, NEW.coletor, NEW.datavenc, NOW(), NEW.valor, NEW.valorpag, NEW.formapag);
    END IF;
END;
```

### 3. Criar o Script PHP
Este script PHP será responsável por ler os registros da tabela `brl_pago` e enviar o recibo de pagamento via WhatsApp e, após o envio, remover os registros da tabela.

Crie um arquivo PHP e adicione o seguinte conteúdo:

```php
<?php
// Configurações do banco de dados
$host = "localhost";
$usuario = "root";
$senha = "vertrigo";
$db = "mkradius";

// Conexão com o banco de dados
$con = mysqli_connect($host, $usuario, $senha, $db);
if (!$con) {
    die("Erro ao conectar ao banco de dados: " . mysqli_connect_error());
}

// Consulta para ler os registros da tabela brl_pago
$query = "SELECT * FROM brl_pago";
$result = mysqli_query($con, $query);

while ($row = mysqli_fetch_assoc($result)) {
    // Extrai os dados do pagamento
    $login = $row['login'];
    $datapag = $row['datapag'];
    $datavenc = $row['datavenc'];
    $valor = $row['valor'];
    $valorpag = $row['valorpag'];
    $coletor = $row['coletor'];
    $formapag = $row['formapag'];

    // Segunda consulta SQL para buscar o número de celular do cliente com base no login
    $cliente = "SELECT celular FROM sis_cliente WHERE login = '$login'";
    $res = mysqli_query($con, $cliente);
    $celular = "";

    // Extrai o número de celular do cliente e aplica a formatação correta
    while ($vreg = mysqli_fetch_row($res)) {
        $celular = formatarNumero($vreg[0]); // Função que formata o número de celular
    }

    // Prepara a mensagem
    $mensagem = "
    *Mensagem Automática de Recebimento de Pagamento*

    *Pagamento recebido em*: $datapag
    *Fatura com vencimento em*: $datavenc
    *Valor da fatura*: R$ $valor
    *Valor do pagamento*: R$ $valorpag
    *Pagamento recebido por*: $coletor
    *Forma de pagamento*: $formapag

    Para segunda via e comprovantes dos pagamentos acesse:
    https://BrLink.org/cliente (coloque o *CPF* do titular)
    ";

    // Envia a mensagem via API do WhatsApp
    enviarMensagemWhatsApp($celular, $mensagem);

    // Após o envio, apaga o registro da tabela brl_pago
    $deleteQuery = "DELETE FROM brl_pago WHERE id = " . $row['id'];
    mysqli_query($con, $deleteQuery);
}

// Função para enviar a mensagem via API do WhatsApp
function enviarMensagemWhatsApp($celular, $mensagem) {
    $apiUrl = 'https://{{baseURL}}/message/sendText/{{instance}}';
    $apiToken = 'seu-token-aqui';

    // Prepara os dados para envio via API
    $data = array(
        "number" => $celular, // Número de celular no formato internacional
        "text" => $mensagem   // Conteúdo da mensagem
    );

    // Converte os dados para JSON
    $jsonData = json_encode($data);

    // Inicializa a sessão cURL
    $ch = curl_init($apiUrl);

    // Configurações do cURL
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, array(
        'Content-Type: application/json',
        'apikey: ' . $apiToken
    ));
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $jsonData);

    // Executa o cURL
    $response = curl_exec($ch);
    curl_close($ch);

    // Verifica a resposta da API
    if ($response) {
        echo "Mensagem enviada com sucesso para o número: $celular \n";
    } else {
        echo "Erro ao enviar mensagem para o número: $celular \n";
    }
}

// Função para formatar o número de celular
function formatarNumero($numero) {
    // Aqui, você pode implementar a formatação adequada para o número de celular no formato internacional
    return $numero;
}

?>
```

### 4. Configurações

#### 4.1 Configurações do Banco de Dados
No início do arquivo PHP, ajuste as configurações do banco de dados mudando do padrão Mk-Auth para os detalhes da sua instalação:

```php
$host = "localhost";    // Host do MySQL
$usuario = "root";      // Usuário do MySQL
$senha = "vertrigo";    // Senha do MySQL
$db = "mkradius";       // Nome do banco de dados
```

#### 4.2 API de WhatsApp
Ajuste as configurações da API de WhatsApp no arquivo PHP:

```php
$apiUrl = 'https://{{baseURL}}/message/sendText/{{instance}}'; // URL da API de WhatsApp
$apiToken = 'seu-token-aqui';                                  // Token da API
```

### 5. Agendamento do Script (Cron Job)
Recomenda-se configurar um cron job no servidor para executar o script PHP periodicamente. Por exemplo, para executar o script a cada 5 minutos, adicione a seguinte linha no arquivo de configuração do cron `(crontab -e)`:

```bash
*/5 * * * * /usr/bin/php /caminho/para/o/script.php
```
