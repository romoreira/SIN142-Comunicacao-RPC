# SIN142 - Sistemas Distribuídos

## Comunicação de Processos com gRPC (Google Remote Procedure Call) 🚀

### Objetivos 🌐

Nesta aula prática de gRPC, os alunos aprenderão os fundamentos da comunicação entre processos distribuídos utilizando chamadas de procedimento remoto (RPC). Serão abordados conceitos como definição de serviços com Protocol Buffers (protobuf), comunicação cliente-servidor, serialização eficiente de dados e os diferentes padrões de comunicação do gRPC (unário, streaming de servidor, streaming de cliente e streaming bidirecional). O objetivo é capacitar os alunos a projetar e implementar serviços distribuídos modernos, amplamente utilizados em arquiteturas de microsserviços.

### Antes de Começar 🌎

* Saber o endereço IP das máquinas envolvidas na comunicação distribuída.

  * IPs: **192.168.1.1** (servidor), **192.168.1.2** (cliente)
* Ter o Python 3.7+ instalado em ambas as máquinas.
* Entender o conceito básico de cliente-servidor.

### Conceitos Fundamentais 📚

|Conceito|Descrição|
|-|-|
|**RPC**|Permite que um programa execute uma função/procedimento em outra máquina como se fosse local.|
|**Protocol Buffers**|Formato de serialização binário do Google, usado para definir a estrutura das mensagens e serviços.|
|**Stub (Client)**|Proxy local que abstrai a comunicação de rede. O cliente chama métodos no stub como se fossem locais.|
|**Servidor gRPC**|Implementa os métodos definidos no `.proto` e escuta conexões dos clientes.|
|**Canal (Channel)**|Conexão de rede entre o cliente e o servidor gRPC.|

### Tipos de Comunicação gRPC 📡

|Tipo|Descrição|Exemplo|
|-|-|-|
|**Unário**|Cliente envia 1 request → Servidor retorna 1 response|Consulta de saldo|
|**Server Streaming**|Cliente envia 1 request → Servidor retorna N responses|Feed de notícias|
|**Client Streaming**|Cliente envia N requests → Servidor retorna 1 response|Upload de dados|
|**Bidirecional**|Cliente e Servidor trocam N mensagens simultaneamente|Chat em tempo real|

\---

### Configuração do Ambiente ☁️

```bash
sudo apt-get update
sudo apt-get install -y python3 python3-pip
pip3 install grpcio grpcio-tools
```

Verificar instalação:

```bash
python3 -c "import grpc; print(grpc.__version__)"
```

\---

### Estrutura do Projeto 📂

```
SIN142-ComunicacaoGRPC/
├── protos/
│   └── calculadora.proto      # Definição do serviço e mensagens
├── calculadora\\\_pb2.py         # Gerado automaticamente (mensagens)
├── calculadora\\\_pb2\\\_grpc.py    # Gerado automaticamente (stubs)
├── servidor.py                # Implementação do servidor gRPC
├── cliente.py                 # Implementação do cliente gRPC
├── makefile                   # Automação de compilação e execução
└── README.md
```

\---

### Passo 1: Definir o Serviço com Protocol Buffers 📋

Crie o arquivo `protos/calculadora.proto`:

```protobuf
syntax = "proto3";

package calculadora;

// Definição das mensagens
message OperacaoRequest {
  double numero1 = 1;
  double numero2 = 2;
}

message ResultadoResponse {
  double resultado = 1;
  string operacao = 2;
}

message NumeroRequest {
  double numero = 1;
}

// Definição do Serviço
service Calculadora {
  // RPC Unário: Cliente envia dois números, servidor retorna o resultado
  rpc Somar (OperacaoRequest) returns (ResultadoResponse);
  rpc Subtrair (OperacaoRequest) returns (ResultadoResponse);
  rpc Multiplicar (OperacaoRequest) returns (ResultadoResponse);
  rpc Dividir (OperacaoRequest) returns (ResultadoResponse);

  // Server Streaming: Cliente envia um número, servidor retorna a tabuada
  rpc Tabuada (NumeroRequest) returns (stream ResultadoResponse);
}
```

\---

### Passo 2: Gerar o Código Python a Partir do `.proto` 🔨

```bash
python3 -m grpc_tools.protoc -I protos/ --python_out=. --grpc_python_out=. protos/calculadora.proto
```

Esse comando gera dois arquivos:

* `calculadora_pb2.py` — Classes das mensagens (OperacaoRequest, ResultadoResponse, etc.)
* `calculadora_pb2_grpc.py` — Classes do servidor e stubs do cliente

\---

### Passo 3: Implementar o Servidor 🖥️

Crie o arquivo `servidor.py`:

```python
import grpc
from concurrent import futures
import calculadora_pb2
import calculadora_pb2_grpc
import time

class CalculadoraServicer(calculadora_pb2_grpc.CalculadoraServicer):
    """Implementacao do servico Calculadora definido no .proto"""

    def Somar(self, request, context):
        resultado = request.numero1 + request.numero2
        print(f"[Servidor] Somar: {request.numero1} + {request.numero2} = {resultado}")
        return calculadora_pb2.ResultadoResponse(resultado=resultado, operacao="soma")

    def Subtrair(self, request, context):
        resultado = request.numero1 - request.numero2
        print(f"[Servidor] Subtrair: {request.numero1} - {request.numero2} = {resultado}")
        return calculadora_pb2.ResultadoResponse(resultado=resultado, operacao="subtracao")

    def Multiplicar(self, request, context):
        resultado = request.numero1 * request.numero2
        print(f"[Servidor] Multiplicar: {request.numero1} * {request.numero2} = {resultado}")
        return calculadora_pb2.ResultadoResponse(resultado=resultado, operacao="multiplicacao")

    def Dividir(self, request, context):
        if request.numero2 == 0:
            context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
            context.set_details("Divisao por zero nao e permitida!")
            return calculadora_pb2.ResultadoResponse()
        resultado = request.numero1 / request.numero2
        print(f"[Servidor] Dividir: {request.numero1} / {request.numero2} = {resultado}")
        return calculadora_pb2.ResultadoResponse(resultado=resultado, operacao="divisao")

    def Tabuada(self, request, context):
        """Server Streaming: retorna a tabuada do numero recebido"""
        print(f"[Servidor] Gerando tabuada do {request.numero}")
        for i in range(1, 11):
            resultado = request.numero * i
            time.sleep(0.5)
            yield calculadora_pb2.ResultadoResponse(
                resultado=resultado,
                operacao=f"{request.numero} x {i}"
            )

def servir():
    servidor = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    calculadora_pb2_grpc.add_CalculadoraServicer_to_server(CalculadoraServicer(), servidor)
    servidor.add_insecure_port('[::]:50051')
    servidor.start()
    print("Servidor gRPC rodando na porta 50051...")
    try:
        servidor.wait_for_termination()
    except KeyboardInterrupt:
        servidor.stop(0)
        print("\nServidor encerrado.")

if __name__ == '__main__':
    servir()
```

\---

### Passo 4: Implementar o Cliente 💻

Crie o arquivo `cliente.py`:

```python
import grpc
import calculadora_pb2
import calculadora_pb2_grpc
import sys
import json

def executar():
    host = sys.argv[1] if len(sys.argv) > 1 else 'localhost'
    canal = grpc.insecure_channel(f'{host}:50051')
    stub = calculadora_pb2_grpc.CalculadoraStub(canal)

    print(f"Conectado ao servidor gRPC em {host}:50051\n")

    print("=" * 40)
    print("COMPARACAO: Protobuf vs JSON")
    print("=" * 40)

    req = calculadora_pb2.OperacaoRequest(numero1=10, numero2=5)
    dados_protobuf = req.SerializeToString()

    dados_json = json.dumps({"numero1": 10, "numero2": 5})
    dados_json_bytes = dados_json.encode("utf-8")

    print(f"Objeto Python: numero1={req.numero1}, numero2={req.numero2}")
    print()
    print(f"--- Protobuf (binario) ---")
    print(f"  Bytes: {dados_protobuf}")
    print(f"  Tamanho: {len(dados_protobuf)} bytes")
    print()
    print(f"--- JSON (texto) ---")
    print(f"  String: {dados_json}")
    print(f"  Tamanho: {len(dados_json_bytes)} bytes")
    print()
    print(f"Protobuf e {len(dados_json_bytes) / len(dados_protobuf):.1f}x menor que JSON!")

    print(f"\n{'=' * 40}")
    print("RPC UNARIO - Operacoes Basicas")
    print("=" * 40)

    response = stub.Somar(calculadora_pb2.OperacaoRequest(numero1=10, numero2=5))
    print(f"Soma: 10 + 5 = {response.resultado}")

    response = stub.Subtrair(calculadora_pb2.OperacaoRequest(numero1=10, numero2=5))
    print(f"Subtracao: 10 - 5 = {response.resultado}")

    response = stub.Multiplicar(calculadora_pb2.OperacaoRequest(numero1=10, numero2=5))
    print(f"Multiplicacao: 10 * 5 = {response.resultado}")

    response = stub.Dividir(calculadora_pb2.OperacaoRequest(numero1=10, numero2=5))
    print(f"Divisao: 10 / 5 = {response.resultado}")

    try:
        response = stub.Dividir(calculadora_pb2.OperacaoRequest(numero1=10, numero2=0))
    except grpc.RpcError as e:
        print(f"Erro esperado (divisao por zero): {e.details()}")

    print(f"\n{'=' * 40}")
    print("SERVER STREAMING - Tabuada")
    print("=" * 40)

    print("Tabuada do 7:")
    responses = stub.Tabuada(calculadora_pb2.NumeroRequest(numero=7))
    for response in responses:
        print(f"  {response.operacao} = {response.resultado}")

    print("\nTodas as operacoes concluidas!")

if __name__ == '__main__':
    executar()
```

\---

### Configuração do Projeto e Execução 📋

Clone o repositório e gere os arquivos necessários:

```bash
git clone https://github.com/romoreira/SIN142-ComunicacaoGRPC.git
cd SIN142-ComunicacaoGRPC
pip3 install grpcio grpcio-tools
python3 -m grpc_tools.protoc -I protos/ --python_out=. --grpc_python_out=. protos/calculadora.proto
```

\---

### Execução e Testes 📔

**Terminal 1 — Iniciar o Servidor (Máquina 192.168.1.1):**

```bash
python3 servidor.py
```

Saída esperada:

```
Servidor gRPC rodando na porta 50051...
```

**Terminal 2 — Executar o Cliente (Máquina 192.168.1.2):**

```bash
python3 cliente.py 192.168.1.1
```

Saída esperada:

```
Conectado ao servidor gRPC em 192.168.1.1:50051

========================================
RPC UNÁRIO - Operações Básicas
========================================
Soma: 10 + 5 = 15.0
Subtração: 10 - 5 = 5.0
Multiplicação: 10 \\\* 5 = 50.0
Divisão: 10 / 5 = 2.0
Erro esperado (divisão por zero): Divisão por zero não é permitida!

========================================
SERVER STREAMING - Tabuada
========================================
Tabuada do 7:
  7.0 x 1 = 7.0
  7.0 x 2 = 14.0
  ...
  7.0 x 10 = 70.0

Todas as operações concluídas!
```

`Neste ponto o cliente deverá ter se comunicado com o servidor remoto via gRPC e recebido todos os resultados.`

\---

### Comparação MPI vs gRPC 🔄

|Característica|MPI|gRPC|
|-|-|-|
|**Paradigma**|Message Passing|Remote Procedure Call|
|**Serialização**|Tipos MPI nativos|Protocol Buffers (binário)|
|**Acoplamento**|Forte (mesmo programa)|Fraco (serviços independentes)|
|**Linguagem**|Geralmente C/C++/Fortran|Multilinguagem (Python, Go, Java, etc.)|
|**Caso de Uso**|HPC, computação científica|Microsserviços, APIs, sistemas web|
|**Descoberta**|host\_file estático|Service discovery, load balancers|
|**Tolerância a Falhas**|Limitada|Deadlines, retries, circuit breakers|

\---

### Proposição de Estudo/Desafio Extra

1. **Client Streaming:** Implemente um método que receba uma sequência de números do cliente (stream) e retorne a média aritmética.
2. **Bidirecional Streaming:** Implemente um chat simples onde cliente e servidor trocam mensagens em tempo real usando streaming bidirecional.
3. **Multilinguagem:** Implemente o cliente em outra linguagem (por exemplo, Go ou Java) e verifique a interoperabilidade com o servidor Python.
4. **Tratamento de Erros:** Adicione interceptors (middlewares) para logging e autenticação nos serviços gRPC.
5. **Benchmark:** Compare a latência e throughput do gRPC com uma implementação equivalente usando REST/JSON (flask + requests).

