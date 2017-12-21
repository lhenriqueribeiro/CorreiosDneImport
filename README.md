# Importa��o do e-DNE dos Correios

Programa console em .NET Core para ler os arquivos da base e-DNE dos Correios e
import�-los em banco de dados com integridade referencial.

O objetivo � permitir a f�cil atualiza��o da base ao somente adicionar os arquivos
delta � medida que eles s�o liberados. Quando um arquivo master for adicionado o
banco � reiniciado.

## Modo de uso

Obtenha o programa:

    git clone https://github.com/vmassuchetto/CorreiosDneImport

Coloque os arquivos do e-DNE sem alterar os nomes originais na pasta `Data`:

    eDNE_Delta_Master_1611.zip
    eDNE_Delta_Master_1612.zip
    eDNE_Delta_Master_1701.zip
    eDNE_Delta_Master_1702.zip
    eDNE_Delta_Master_1703.zip
    eDNE_Delta_Master_1704.zip
    eDNE_Master_1611.zip

Execute o programa informando a conex�o com o banco de dados:

    dotnet run "Server=Servidor;Database=Banco;User Id=Usuario;Password=Senha;"

## Funcionamento

A importa��o � incremental usando arquivos `Master` -- completos e liberados
anualmente, e `Delta` -- parciais e liberados mensalmente. Arquivos `Delta`
possuem um indicador de opera��o para cada registro (`INS`, `UPD` e `DEL`).

Arquivos importados s�o registrados em uma tabela de controle para que n�o
serem lidos novamente. Descompacta��o do ZIP � feita em um diret�rio tempor�rio
do sistema que � apagado ap�s a importa��o.

Independente da quantidade de arquivos na pasta `Data`, o programa ir�
construir a ordem correta a partir da �ltima importa��o `Master` presente ou
que j� foi importada anteriormente.

Arquivos que n�o estejam em sequ�ncia n�o ser�o importados. Registros j�
importados s�o ignorados.

As tabelas s�o [mapeadas](https://github.com/vmassuchetto/CorreiosDneImport/blob/master/CorreiosDneImport/DneImport.php#L44-L62)
da seguinte forma:

    LOG_LOGRADOURO -> CorreiosLogradouro
    LOG_LOCALIDADE -> CorreiosLocalidade
    LOG_FAIXA_CPC  -> CorreiosFaixaCpc

Os campos s�o convertidos para CamelCase:

    LOC_NU         -> LocNu
    UFE_SG         -> UfeSg
    LOG_NO_ABREV   -> LogNoAbrev_

Execu��o:

![Indica��o de progresso da importa��o](Assets/Progresso.jpg)

## Precau��es

A integridade de dados � desligada durante o processo de importa��o. Execute a
importa��o em ambiente de testes para ter certeza que a sequ�ncia de deltas
liberada pelos Correios n�o invalida a integridade referencial. Estes casos
podem acontecer e lan�am esta exce��o ao final da importa��o:

    Unhandled Exception: System.Data.SqlClient.SqlException:
        The ALTER TABLE statement conflicted with the FOREIGN KEY constraint "FK_CorreiosVarLoc_CorreiosLocalidade".
        [...], table "dbo.CorreiosLocalidade", column 'LocNu'

No caso acima, algum valor do campo `LocNu` em `CorreiosVarLoc` n�o
consta em `CorreiosLocalidade`. Para encontrar este valor:

    SELECT vl.LocNu, l.LocNu
    FROM CorreiosVarLoc vl
    LEFT JOIN CorreiosLocalidade l ON l.LocNu = vl.LocNu
    WHERE l.LocNu IS NULL

Resultado:

    LocNu	LocNu
    7261	NULL
    7261	NULL
    7261	NULL
    7261	NULL
    7261	NULL
    7261	NULL

Para corrigir:

    DELETE FROM CorreiosVarLoc WHERE LocNu = 7261