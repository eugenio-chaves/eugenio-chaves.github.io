[English version](https://eugenio-chaves.github.io/blog/2022/creating-a-custom-wazuh-integration)

## Integrando Wazuh com Discord

### O Motivo

No futuro, eu quero monitorar minha rede doméstica e o resto do meu ambiente com Wazuh integrado com Suricata. Também quero uma forma fácil para visualizar os alertas que o Wazuh irá gerar.

![](/docs/assets/images/07.png)

Eu poderia usar as integrações prontas do Wazuh, como a do [Slack](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-integrate-slack.html), por exemplo. Decidi usar o Discord porque é um aplicativo que uso praticamente o dia todo e um exemplo perfeito para demonstrar como criar uma integração customizada do Wazuh com apps externos.

É possível criar uma integração customizada do Wazuh para enviar seus alertas para qualquer lugar que tenha um webhook ou alguma forma de receber dados via POST requests. Como Microsoft Teams, por exemplo.

### O Ambiente

Eu estou assumindo que você já esteja com o Wazuh instalado e funcionando. Em meu ambiente atual, estou com uma instalação "All in One" na minha VM kali, você pode seguir com qualquer distribuição Linux que quiser. Se você estiver usando o Wazuh distribuído em clusters, irá precisar replicar toda esta configuração nas managers onde você queira que a integração funcione.

### Criando o Webhook

A primeira coisa a se fazer é criar um webhook no chat do **seu** servidor de Discord
```markdown
- Abra as configurações de seu servidor
- Na aba "Integrações", clique em "webhooks" para gerar um
- Salve o link por enquanto
```

### Criando a Integração

Entre na pasta de integração do Wazuh. Você deve ver os scripts de integrações padrões
```bash
cd /var/ossec/integrations 
```
![](/docs/assets/images/01.png)

Como você pode ver, existem dois scritps do slack, um em bash e outro em python, a razão para isso é porque o script em bash vai funcionar como um launcher para o script em python, que é o core da integração.

Copie os dois e passe o nome para **custom-discord**

- Todas as integrações customizadas do Wazuh precisam que o nome inicie com **custom-**
```bash
cp slack custom-discord
cp slack.py custom-discord.py
```

Os scritps do **Slack** foram feitos pelo time do Wazuh para integrar com um canal de Slack via webhook, podemos modificar o script em python para integrar com o Discord. Para isso, a única modificação que será preciso fazer é na função **generate_msg()** dentro do script **custom-discord.py**.

Antes de começar, acredito que seria interessante demonstrar como uma integração é chamada e como você pode controlar quais os tipos de alertas do Wazuh que podem disparar a integração.

Abra o arquivo de configuração da manager do Wazuh **ossec.conf**
```bash
vim /var/ossec/etc/ossec.conf
```

Este bloco de xml é o que ativa a integração, você vai precisar colocar esse bloco no arquivo de configuração da manager do Wazuh, **ossec.conf**. Você pode colocar em qualquer posição dentro do arquivo, só tome cuidado para não inserir no meio de um bloco de outra configuração.
```xml
  <integration>
    <name>custom-discord</name>
    <hook_url>https://discord.com/api/webhooks/hook</hook_url>
    <level>7</level>
    <alert_format>json</alert_format>
  </integration>
``` 
![](/docs/assets/images/02.png)

- A Tag **name** na linha 354 é onde você define o nome do arquivo que irá executar o script em python.
- A Tag **hook** é onde você deve inserir seu webhook.
- Na linha 356, você pode controlar qual condição irá chamar a integração, no meu caso, qualquer alerta do Wazuh onde o nível seja maior o igual a 07.

Você pode usar as seguintes condições para disparar a integração:
```xml
<group>suricata,sysmon</group> Somente as regras desses grupos vão disparar a integração.
<level>12</level> Somente regras de nível igual ou maior que 12 irão disparar a integração.
<rule_id>1299,1300</rule_id> Somente as regras com esses IDs irão irão disparar a integração.
```

### Customizando o Script

Após ativar a integração, o próximo passo seria customizar o script custom-discord.py

Abra o script com seu editor de texto favorito, na linha **76** você deve substituir a função **generate_msg()** do script com a função abaixo;

Essa função irá pegar um alerta do Wazuh como argumento e como os alertas estão chegando em formato json, tudo que precisa ser feito é preencher os valores das chaves dentro do dicionário da função e enviar para o webhook do Discord.

Quando eu construí esse payload, usei o repositório do [Birdie0](https://github.com/Birdie0) para me ajudar a entender como customizar o que posso enviar para o webhook do Discord.

Eu recomendo que você cheque o repositório do Birdie [birdie0.github.io/discord-webhooks-guide/index.html](https://birdie0.github.io/discord-webhooks-guide/index.html) e tente achar se tem algum formato diferente que você gostaria de usar em seu alerta.

Uma boa ferramenta para ajudar com isto é o **Postman** https://www.postman.com/
```python
def generate_msg(alert):
	#save the rule level
    level = alert['rule']['level']
    #compare rules level to set colors of the alert
    if (level <= 4):
    	#green
        color = "3731970"
    elif (level >= 5 and level <= 12):
        #yellow
        color = "15919874"
    else:
        #red
        color = "15870466"

    if 'agentless' in alert:
        agent_ = 'agentless'
    else:
        agent_ = alert['agent']['name']
    #data that the webhook will receive and use to display the alert in discord chat
    payload = json.dumps({
      "embeds": [
        {
          "title": "Wazuh Alert - Rule {}".format(alert['rule']['id']),
          "color": "{}".format(color),
          "description": "{}".format(alert['rule']['description']),
          "fields": [
            {
              "name": "Agent",
              "value": "{}".format(agent_),
              "inline": True
            },
            {
              "name": "Location",
              "value": "{}".format(alert['location']),
              "inline": True
            },
            {
            "name": "Rule Level",
            "value": "{}".format(alert['rule']['level']),
            "inline": True
            }
          ]
        }
      ]
    })

    return payload
```
Este é o formato do alerta:

![](/docs/assets/images/03.png)

Eu também gosto de alterar a função **debug()**, com isso consigo controlar mais livremente os logs do script.

O caminho completo para o arquivo de logs é este:
```
/var/ossec/logs/integrations.log
```
Você pode ativar e desativar os logs com a função **deb**
```python
def debug(msg):
        # debug log
        deb = True
        if deb == True:
            msg = "{0}: {1}\n".format(now, msg)
            print(msg)
            f = open(log_file, "a")
            f.write(msg)
            f.close()
```
Você deve alterar as permissões e owners dos arquivos:
```bash
chmod 750 custom-*
chown root:wazuh custom-*
```
Você também vai precisar do módulo requests:
```bash
pip3 install requests
```

### Criando uma Regra Customizada

Tudo deve estar pronto agora, mas antes de reiniciar a manager do Wazuh e ativar a integração, eu gosto de criar uma regra onde eu posso ativar ela manualmente para testar a integração.

Crie um arquivo no diretório **/var/log** chamado **test.log**
```bash
touch /var/log/test.log
```
Abra o arquivo ossec,conf novamente e vá para o final do arquivo, você deve ver vários blocos com o nome de **localfile**, eles indicam um arquivo onde o Wazuh irá coletar logs.

Insira esse bloco no final do arquivo:
```xml
  <localfile>
    <location>/var/log/test.log</location>
    <log_format>syslog</log_format>
  </localfile>
```
Agora crie uma regra, você vai precisar abrir o arquivo apropriado, **local_rules.xml**

Criando sua regra nesse arquivo ela não será perdida durante um futuro update do Wazuh.
```bash
vim /var/ossec/etc/rules/local_rules.xml
```
Acrescente a seguinte regra no arquivo e salve ele:
```xml
  <rule id="119999" level="12">
    <regex>^test$</regex>
    <description>Test rule to configure integration</description>
  </rule>
```
Agora, sempre que você inserir a palavra **test** no arquivo **/var/log/test.log** a regra deve ser disparada, como minha trigger da integração é qualquer regra maior ou igual a 07, isso será o suficiente para testar a integração;

Para testar a regra, existe o binário que é exclusivo para isso, **wazuh-logtest**

Rode o binário e digite a palavra **test**, você deverá ver sua regra sendo disparada.
```bash
/var/ossec/bin/wazuh-logtest
```

![](/docs/assets/images/04.png)

Reinicie a manager e dispare o alerta, você deverá receber um alerta no chat do seu servidor do Discord.
```bash
/var/ossec/bin/wazuh-control restart
echo -e "test" /var/log/test.log
```
O alerta no Discord:

![](/docs/assets/images/05.png)

### Debugando

Se encontrar problema, os arquivos de logs que podem ajudar são: 
- /var/ossec/logs/ossec.log
- /var/ossec/logs/integration.log

Você também pode fazer o daemon [integrator](https://documentation.wazuh.com/current/user-manual/reference/daemons/wazuh-integratord.html) mais verboso para ajudar com o debugging, para isso, execute ele com a flag **-d** ou **-dd**
```bash
/var/ossec/bin/wazuh-integratord  -d
```
Depois disso, os logs no arquivo ossec.log relacionados a integração devem trazer mais informações, ele deverá mostrar todo seu fluxo de execução.

### Conclusão

Primeiramente eu quero agradecer ao Alexandre Borges [@ale_sp_brazil](https://twitter.com/ale_sp_brazil) por me incentivar a iniciar este blog. Você definitivamente deveria checar o blog dele [exploitreversing.com](https://exploitreversing.com/)

A primeira vez que precisei criar uma integração customizada, não achei muitos materiais disponíveis falando a respeito deste assunto, espero que isso possa ajudar alguém.

[Topo](https://eugenio-chaves.github.io/blog/2022/wazuh-criando-uma-integracao-customizada)
