# SteoFunctions
Segundo desafio do Bootcamp Code Girls consistem em utilizar o Step Functions da AWS e mapear o passo a passo da utilização do mesmo

Passo a Passo – Utilização do AWS Step Functions
Na página Orquestração de fluxos de trabalho com tecnologia sem servidor | AWS Step Functions | Amazon Web Services você encontra uma explicação detalhada sobre o que é o Step Functions, como funciona, seus benefícios, casos de uso e serviços relacionados.
Abaixo, segue um resumo simplificado para quem não é familiarizado com o serviço.
O que é o Step Functions
O AWS Step Functions é um serviço de fluxo de trabalho visual que ajuda os desenvolvedores a usar os serviços da AWS para criar aplicações distribuídas, automatizar processos, orquestrar microsserviços e criar pipelines de dados e machine learning (ML).
Como funciona (resumido)
1.	Definição do fluxo: Crie seu fluxo de trabalho de forma visual no Workflow Studio ou por código, no console ou no VS Code.
2.	Configuração dos recursos: Especifique quais serviços (ex.: AWS Lambda, Amazon SNS) serão acionados em cada etapa.
3.	Execução e teste: Inicie a execução, acompanhe as etapas em tempo real e depure falhas diretamente no console.
Benefícios e Atributos
•	Tratamento de erros automático
•	Histórico detalhado de execução
•	Escalabilidade automática
•	Alta disponibilidade
•	Pagamento sob demanda
•	Segurança integrada

AWS Step Functions – Comece a usar
1.	Apos o login acesse a pagina Step Functions | us-east-2
 
2.	Crie seu fluxo do zero
 
3.	Crie a máquina de estado informando:
Nome da máquina: desafioDIO
Tipo: Padrão
 

Conforme a arquitetura definida no desafio anterior (e-commerce) arraste os itens necessários para montar o fluxo.
 

Resultado final JSON:
{
  "Comment": "Fluxo de compra e-commerce com SQS, SNS, Lambda e S3",
  "StartAt": "ReceberPedido",
  "States": {
    "ReceberPedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "https://sqs.us-east-1.123456789012.amazonaws.com/MinhaFilaPedidos",
        "MessageBody": {
          "pedido": "detalhes-do-pedido"
        }
      },
      "Next": "ValidarPedido"
    },
    "ValidarPedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ValidarPedido",
        "Payload": {
          "pedido.$": "$"
        }
      },
      "ResultPath": "$.pedidoValidado",
      "Next": "PedidoAprovado?"
    },
    "PedidoAprovado?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pedidoValidado.aprovado",
          "BooleanEquals": true,
          "Next": "NotificarPedidoOK"
        }
      ],
      "Default": "NotificarFalhaPedido"
    },
    "NotificarFalhaPedido": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:PedidosFalha",
        "Message": "Pedido não aprovado"
      },
      "End": true
    },
    "NotificarPedidoOK": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:PedidosSucesso",
        "Message": "Pedido aprovado"
      },
      "Next": "ProcessarPagamento"
    },
    "ProcessarPagamento": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ProcessarPagamento",
        "Payload": {
          "pedido.$": "$"
        }
      },
      "ResultPath": "$.pagamento",
      "Next": "PagamentoAprovado?"
    },
    "PagamentoAprovado?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.pagamento.aprovado",
          "BooleanEquals": true,
          "Next": "NotificarPagamentoAprovado"
        }
      ],
      "Default": "NotificarFalhaPagamento"
    },
    "NotificarFalhaPagamento": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:PagamentosFalha",
        "Message": "Pagamento não aprovado"
      },
      "End": true
    },
    "NotificarPagamentoAprovado": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:PagamentosSucesso",
        "Message": "Pagamento aprovado"
      },
      "Next": "IntegrarBackstage"
    },
    "IntegrarBackstage": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "IntegrarBackstage",
        "Payload": {
          "pedido.$": "$"
        }
      },
      "Next": "GerarComprovanteS3"
    },
    "GerarComprovanteS3": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "GerarComprovante",
        "Payload": {
          "pedido.$": "$"
        }
      },
      "ResultPath": "$.comprovante",
      "Next": "NotificarPedidoFinal"
    },
    "NotificarPedidoFinal": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:PedidosFinal",
        "Message": {
          "default": "Pedido confirmado e comprovante disponível",
          "s3Url.$": "$.comprovante.s3Url"
        }
      },
      "End": true
    }
  }
}

 
O que esse fluxo faz
1.	ReceberPedido → envia para SQS.
2.	ValidarPedido (Lambda) → verifica se o pedido é válido.
3.	PedidoAprovado? → decisão (Choice).
o	Se não aprovado → envia SNS de falha.
o	Se aprovado → continua.
4.	ProcessarPagamento (Lambda) → chama gateway de pagamento.
5.	PagamentoAprovado? → decisão (Choice).
o	Se não aprovado → envia SNS de falha.
o	Se aprovado → continua.
6.	IntegrarBackstage (Lambda) → envia info ao sistema interno.
7.	GerarComprovanteS3 (Lambda) → gera arquivo/PDF, salva no S3 e retorna o link.
8.	NotificarPedidoFinal (SNS) → envia mensagem ao cliente com o link do comprovante no S3.


