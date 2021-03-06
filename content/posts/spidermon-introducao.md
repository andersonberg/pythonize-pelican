---
title: Como monitorar web-crawlers com Spidermon
date: 2020-09-06 00:00
tags: ["python", "web-crawlers", "web-spiders", "spidermon", "spider"]
category: python
slug: como-monitorar-spiders-spidermon
meta_description: Aprendendo a utilizar o Spidermon para monitorar web-crawlers
---


Uma das etapas mais importantes do web-scraping (ou raspagem de dados) é garantir a qualidade das informações extraídas. Devido ao grande volume de dados que podem ser coletados por um spider, é bem complicado analisar tudo manualmente. Porém existem ferramentas para auxiliar nesse processo que vão diminuir muito o trabalho de quem faz esse tipo de análise. Uma dessas ferramentas é o Spidermon, que vou apresentar neste artigo.


Spidermon é uma extensão do Scrapy usado para monitorar spiders. É um framework bem robusto e, pra mim, indispensável nos projetos que trabalho. Com ele é possível validar a execução ou os resultados produzidos por spiders. Além disso, você pode criar relatórios e enviar notificações via slack ou e-mail, por exemplo.

Para poder ver o Spidermon em ação, vamos começar criando um spider bem simples. Vamos supor que queremos extrair título do livro e respectivo ranking da seguinte página: https://www.goodreads.com/list/show/3.Best_Science_Fiction_Fantasy_Books/

Nosso spider seria assim:

```python
class BooksSpider(scrapy.Spider):
    name = 'books'
    allowed_domains = ['goodreads.com']
    start_urls = [
        'https://www.goodreads.com/list/show/3.Best_Science_Fiction_Fantasy_Books/'
    ]

    def parse(self, response):
        books_list = response.css(
            'tr[itemtype="http://schema.org/Book"]')
        for book in books_list:
            yield{
                'title': book.css(
                    '.bookTitle span[itemprop="name"]::text'
                    ).get(),
                'rating': float(
                    book.css('.minirating::text'
                    ).re_first(r'\d[\d.]*'))
            }

        yield scrapy.Request(
            response.urljoin(
                response.css('.next_page::attr(href)').get()
            )
        )
```


Antes de criar o primeiro monitor, habilite o Spidermon nas configurações do projeto. É bem simples, adicione as seguintes linhas no arquivo settings.py do seu projeto scrapy:

```python
SPIDERMON_ENABLED = True
EXTENSIONS = {
    'spidermon.contrib.scrapy.extensions.Spidermon': 500,
}
```


## Criando monitores

Agora para que o Spidermon comece a monitorar os resultados, precisamos criar monitores. Monitores no Spidermon são baseados no unittest, portanto são basicamente casos de teste executados em determinado momento da execução do spider. Os monitores por sua vez, podem ser agrupados em um MonitorSuite. As classes MonitorSuite são importantes para definir quais monitores serão executados e quais ações terão lugar antes e depois que o grupo de monitores terminar seu trabalho.

Vamos entender melhor com um exemplo: Queremos garantir que o spider vai extrair pelo menos 10 itens. Através dos stats finais do spider, temos acesso a um atributo chamado `item_scraped_count`, que mostra quantos itens o spider conseguiu extrair. E aí você pode usar o `assertTrue` para comparar os valores:

```python
from spidermon import Monitor, MonitorSuite, monitors

@monitors.name('Item count')
class ItemCountMonitor(Monitor):

    @monitors.name('Minimum number of items')
    def test_minimum_number_of_items(self):
        item_extracted = getattr(
            self.data.stats, 'item_scraped_count', 0)
        minimum_threshold = 10

        msg = 'Extracted less than {} items'.format(
            minimum_threshold)
        self.assertTrue(
            item_extracted >= minimum_threshold, msg=msg
        )

class SpiderCloseMonitorSuite(MonitorSuite):

    monitors = [
        ItemCountMonitor,
    ]
```

Crie um arquivo chamado monitors.py e salve esse código.

A ideia é que este MonitorSuite execute após o spider terminar a execução, então o próximo passo é incluir mais uma configuração no settings.py:

```python
SPIDERMON_SPIDER_CLOSE_MONITORS = (
    'tutorial.monitors.SpiderCloseMonitorSuite',
)
```

Agora é só executar o spider normalmente para visualizar os monitores no log.


## Validando itens

Uma das principais formas de garantir a qualidade dos dados extraídos é validá-los segundo um modelo. Este modelo pode ser definido em um (JSON schema)[https://json-schema.org/]. Entre outras coisas é possível definir os campos obrigatórios dos itens ou usar uma expressão regular para validar o conteúdo de um campo.

Antes de utilizar a validação com o JSON schema, é preciso instalar o pacote `jsonschema`, se já não estiver instalado:

```bash
    $ pip install jsonschema
```

Crie um arquivo chamado schema.json na raiz do projeto ou num diretório específico para esses arquivos (só a nível de organização). Nele insira o seguinte código:

```yaml
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
        "title":{
            "type": "string"
        },
        "rating":{
            "type": "number"
        }
    },
    "required":[
        "title",
        "rating"
    ]
}
```

Mais algumas configurações são necessárias serem adicionadas ao settings.py. Primeiro, habilite o `item pipeline` e depois adicione o caminho do arquivo criado anteriormente no `SPIDERMON_VALIDATION_SCHEMAS`:

```python
ITEM_PIPELINES = {
    'spidermon.contrib.scrapy.pipelines.ItemValidationPipeline': 800,
}
SPIDERMON_VALIDATION_SCHEMAS = [
    '/home/anderson/code/crawler_test_spidermon/crawler_test_spidermon/schema.json'
]
```

O Spidermon vai exibir os erros de `schema` nos `stats` ao final da execução do spider. Se preferir, você pode adicionar os erros de validação nos próprios itens, isto pode facilitar na hora de encontrar o item que está fora do padrão definido. Para isso adicione o seguinte no settings.py:

```python
SPIDERMON_VALIDATION_ADD_ERRORS_TO_ITEMS = True
```

## Enviando notificações via Telegram

Esta é outra função bem interessante do Spidermon. Você não precisa ficar checando um spider toda vez que ele finaliza a execução para poder ver se houve algum erro. O Spidermon pode enviar notificações via Email, Slack, Telegram entre outros. Neste artigo vou cobrir o envio de mensagens pelo Telegram, mas a documentação do Spidermon é bem completa com relação aos outros.

Para usar essa funcionalidade, você precisa de um (token de bot)[https://core.telegram.org/bots] do Telegram e adicionar as seguintes opções no settings.py:

```python
SPIDERMON_TELEGRAM_SENDER_TOKEN = '<TELEGRAM_SENDER_TOKEN>'
SPIDERMON_TELEGRAM_RECIPIENTS = ['chatid', '@channelname']
```

A opção `SPIDERMON_TELEGRAM_RECIPIENTS` contém todos os destinatários que irão receber as notificações do Spidermon.

Em seguida, no arquivo monitors.py, importe e adicione a classe que executa a ação de envio de mensagem ao Telegram:

```python
from spidermon.contrib.actions.telegram.notifiers import SendTelegramMessageSpiderFinished
...

class SpiderCloseMonitorSuite(MonitorSuite):

    monitors = [
        ItemCountMonitor,
    ]
    monitors_failed_actions = [SendTelegramMessageSpiderFinished]
```

Pronto, se houver algum erro definido nos monitores, a notificação será enviada via Telegram para todos os destinatários.