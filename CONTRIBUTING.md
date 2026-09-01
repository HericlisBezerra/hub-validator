# Contribuindo

Esta skill é o playbook do time da HeBeDigital para tirar um mini-SaaS do PRD até a entrega ao cliente.

## O que faz uma boa contribuição

**A ordem dos 8 passos não se negocia sem argumento.** Cada passo existe porque o seguinte destrói a chance de fazê-lo depois. Se você quer mudar a sequência, mostre o caso concreto em que ela falhou — não a intuição de que ficaria melhor.

**Achado de segurança precisa de evidência, não de teoria.** A query que revela, o objeto que passa despercebido, o caminho que um atacante percorre. "Talvez seja inseguro" não entra.

**Prompt novo para a Lovable precisa passar em dois testes:** pede evidência em vez de opinião, e resiste a resposta complacente. Modelo gerador é otimista sobre o próprio trabalho — prompt que aceita "está tudo certo" não vale nada.

## O que será recusado

- Outra checklist de segurança. A `hebe-security-scan` já tem 17 categorias; esta skill é outro **veículo** (prompts para colar na Lovable), não outro conteúdo.
- Passo que não diz **quando** roda. A skill inteira gira em torno de ordem.
- Token ou padrão de design inventado. O que não está no design system não existe — declare a lacuna.
- Script ou dependência. Isto é instrução, e revisar não pode exigir executar.

## Licença

Contribuindo, você aceita que seu trabalho saia sob a licença MIT deste repositório.
