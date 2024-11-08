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
    envio TINYINT(1) NOT NULL DEFAULT 0,
    PRIMARY KEY (id)
);
```

### 2. Criar a Trigger no Banco de Dados
Agora, crie uma trigger que será ativada após a atualização do status de pagamento na tabela `sis_lanc`. A trigger insere automaticamente os dados na tabela `brl_pago`.

Execute o seguinte comando SQL para criar a trigger:

```sql
DELIMITER $$

CREATE TRIGGER `tig_brl_pag` AFTER UPDATE ON `sis_lanc`
FOR EACH ROW
BEGIN
    -- Declaração de variáveis para armazenar informações do cliente
    DECLARE var_cli_ativado VARCHAR(1);
    DECLARE var_zap VARCHAR(3);

    -- Verifica se o status foi atualizado para "pago"
    IF NEW.status = 'pago' THEN

        -- Busca os valores de 'cli_ativado' e 'zap' na tabela 'sis_cliente'
        SELECT cli_ativado, zap INTO var_cli_ativado, var_zap
        FROM sis_cliente
        WHERE login = NEW.login
        LIMIT 1;

        -- Verifica se o cliente está ativo e deseja receber mensagens via WhatsApp
        IF var_cli_ativado = 's' AND var_zap = 'sim' THEN

            -- Verifica se o ID já existe na tabela brl_pago para evitar duplicidades
            IF NOT EXISTS (SELECT 1 FROM brl_pago WHERE id = NEW.id) THEN

                -- Insere os dados na tabela auxiliar brl_pago
                INSERT INTO brl_pago (id, login, coletor, datavenc, datapag, valor, valorpag, formapag)
                VALUES (NEW.id, NEW.login, NEW.coletor, NEW.datavenc, NEW.datapag, NEW.valor, NEW.valorpag, NEW.formapag);

            END IF;
        END IF;
    END IF;
END$$

DELIMITER ;
```

### 3. Criar o Script PHP
Este script PHP será responsável por ler os registros da tabela `brl_pago`, enviar o recibo de pagamento via WhatsApp e, após o envio, marcar o registro como enviado, atualizando o campo `envio` para `1`, permitindo manter um histórico dos envios.

Crie um arquivo PHP e adicione o seguinte conteúdo:

```php
<?php
// Configurações do banco de dados
$host = "localhost";
$usuario = "root";
$senha = "vertrigo";
$db = "mkradius";

// Configurações da API
$apiUrl = 'https://{{baseURL}}/message/sendText/{{instance}}'; // URL da API
$apiToken = 'seu-token-aqui'; // Token da API

// Conexão com o banco de dados
$con = new mysqli($host, $usuario, $senha, $db);
if ($con->connect_error) {
    die("Erro ao conectar ao banco de dados: " . $con->connect_error);
}

// Consulta para ler registros não enviados da tabela brl_pago
$query = "SELECT * FROM brl_pago WHERE envio = 0";
$stmt = $con->prepare($query);
$stmt->execute();
$result = $stmt->get_result();

while ($row = $result->fetch_assoc()) {
    // Extrai os dados do pagamento e formata as datas
    $login = $row['login'];
    $datapag = date('d/m/Y', strtotime($row['datapag'])); // Formata a data de pagamento para dd/mm/aaaa
    $datavenc = date('d/m/Y', strtotime($row['datavenc'])); // Formata a data de vencimento para dd/mm/aaaa
    $valor = number_format($row['valor'], 2, ',', '.'); // Formata o valor no padrão brasileiro R$ 1.234,56
    $valorpag = number_format($row['valorpag'], 2, ',', '.'); // Formata o valor pago no padrão brasileiro
    $coletor = $row['coletor'];
    $formapag = $row['formapag'];

    // Consulta SQL para buscar o número de celular do cliente com base no login usando prepared statements
    $clienteQuery = "SELECT celular FROM sis_cliente WHERE login = ?";
    $clienteStmt = $con->prepare($clienteQuery);
    $clienteStmt->bind_param('s', $login); // "s" indica que o parâmetro é uma string
    $clienteStmt->execute();
    $clienteResult = $clienteStmt->get_result();
    $celular = "";

    // Extrai o número de celular do cliente e aplica a formatação correta
    if ($clienteRow = $clienteResult->fetch_assoc()) {
        $celular = formatarNumero($clienteRow['celular']);
    }

    // Prepara a mensagem sem espaços desnecessários
    $mensagem = "*Mensagem Automática de Recebimento de Pagamento*\n\n";
    $mensagem .= "*Pagamento recebido em*: $datapag\n";
    $mensagem .= "*Fatura com vencimento em*: $datavenc\n";
    $mensagem .= "*Valor da fatura*: R$ $valor\n";
    $mensagem .= "*Valor do pagamento*: R$ $valorpag\n";
    $mensagem .= "*Pagamento recebido por*: $coletor\n";
    $mensagem .= "*Forma de pagamento*: $formapag\n\n";
    $mensagem .= "Para segunda via e comprovantes dos pagamentos acesse:\n";
    $mensagem .= "https://BrLink.org/cliente (coloque o *CPF* do titular)\n";

    // Envia a mensagem via API do WhatsApp
    if (enviarMensagemWhatsApp($celular, $mensagem)) {
        // Marca o registro como enviado na tabela brl_pago
        $updateQuery = "UPDATE brl_pago SET envio = 1 WHERE id = ?";
        $updateStmt = $con->prepare($updateQuery);
        $updateStmt->bind_param('i', $row['id']); // "i" indica que o parâmetro é um inteiro
        $updateStmt->execute();
    }
}

// Função para enviar a mensagem via API do WhatsApp
function enviarMensagemWhatsApp($celular, $mensagem) {
    global $apiUrl, $apiToken;

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
        return true;
    } else {
        echo "Erro ao enviar mensagem para o número: $celular \n";
        return false;
    }
}

// Função para formatar o número de celular
function formatarNumero($numero) {
    // Remove todos os caracteres que não sejam números
    $numero = preg_replace('/\D/', '', $numero);

    // Verifica se o número tem o código de área (DDD) com 2 dígitos e o número com 8 ou 9 dígitos
    if (strlen($numero) == 10) {
        // Número de telefone com 8 dígitos (sem o 9 na frente)
        $numero = '55' . substr($numero, 0, 2) . '9' . substr($numero, 2); // Adiciona o DDI 55 e insere o 9 antes do número
    } elseif (strlen($numero) == 11) {
        // Número de telefone com 9 dígitos (formato correto)
        $numero = '55' . $numero; // Adiciona o DDI 55
    }

    return $numero;
}
?>
```

### 4. Configurações

#### 4.1 Configurações do Banco de Dados
No início do arquivo PHP, ajuste as configurações do banco de dados mudando do padrão Mk-Auth para os detalhes da sua instalação:

```php
// Configurações do banco de dados
$host = "localhost";
$usuario = "root";
$senha = "vertrigo";
$db = "mkradius";
```

#### 4.2 API de WhatsApp
Ajuste, também, no início do arquivo PHP as configurações da Evolution API de WhatsApp no arquivo PHP:

```php
// Configurações da API
$apiUrl = 'https://{{baseURL}}/message/sendText/{{instance}}'; // URL da API
$apiToken = 'seu-token-aqui'; // Token da API
```

### 5. Agendamento do Script (Cron Job)
Recomenda-se configurar um cron job no servidor para executar o script PHP periodicamente. Por exemplo, para executar o script a cada minuto, adicione a seguinte linha no arquivo de configuração do cron `crontab -e` ou globalmente com usuário root editando o arquivo `/etc/crontab`:

```bash
* * * * * root /opt/php7/bin/php -q /opt/mk-auth/scripts/brl_pag.php >/dev/null 2>&1
```

Se desejar executar o script a cada 5 minutos:

```bash
*/5 * * * * root /opt/php7/bin/php -q /opt/mk-auth/scripts/brl_pag.php >/dev/null 2>&1
```
