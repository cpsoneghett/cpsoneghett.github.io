---
title:  "Padrões de Design (Design Patterns) com Java - Parte 1: Strategy e Singleton"
date:   2023-05-26 15:04:23
categories: [post]
tags: [design, patterns, java, strategy, singleton]
---

## Introdução

Este post está sendo inspirado por um sábado chuvoso, após uma entrevista malsucedida durante a semana. Fiquei me sentindo mal por ter ficado nervoso e ter esquecido coisas básicas que já vinha implementando há vários anos. Assim, como acontece toda vez que questiono minha capacidade, sentei e voltei a estudar aquilo que (achava que) já sabia, além de reimplementar o problema proposto. Escrevo isso aqui sem nem a intenção de que alguém leia, mas apenas para fixar meu conhecimento. O tema em questão trata de padrões de design de desenvolvimento com Java, os famosos "Design Patterns".

Neste post, vou começar falando um breve resumo sobre o que são esses padrões, algumas situações/problemas computacionais e padrões que resolvam estes problemas.

Spoiler: os padrões que utilizarei serão o Strategy e o Singleton.

<div id='design-patterns-definition' />

## 1. O que são Design Patterns?

> "Design Patterns descrevem soluções simples e elegantes para problemas específicos no design de software orientado a objetos. Os padrões de design capturam soluções que foram desenvolvidas e evoluíram ao longo do tempo."
>
> Fonte: [Livro "Design Patterns - Elements of Reusable Object-Oriented Software" (1994)](https://github.com/TushaarGVS/Design-Patterns-Mentorship/blob/master/Erich%20Gamma%2C%20Richard%20Helm%2C%20Ralph%20Johnson%2C%20John%20M.%20Vlissides-Design%20Patterns_%20Elements%20of%20Reusable%20Object-Oriented%20Software%20%20-Addison-Wesley%20Professional%20(1994).pdf)

Em outras palavras, de acordo com a citação que se encontra na introdução do livro acima (que é uma referência para o estudo do mesmo), são padrões de desenvolvimento que estabelecem algumas alternativas de soluções para problemas existentes na programação e que, inclusive, podem ser adaptados para cada problema. Assim como em diversas áreas do conhecimento, sejam nas ciências humanas e da natureza, nas linguagens e qualquer outra, existem técnicas e soluções para determinados contextos a serem tratados, seja para resolver um problema ou para melhorar uma condição já existente. Dito isso, vamos para o problema.

## 2. O problema: [Balanceador de Carga¹](https://pt.wikipedia.org/wiki/Balanceamento_de_carga) (Load Balancer) 

`Primeiro, um disclaimer: Minha intenção aqui especificamente nesse post tratar detalhes da linguagem Java, embora eu estou tentando trazer uma explicação do que são Padrões de Design de uma forma geral, até para quem não tem contato com programação. Mas saber um pouco de lógica de programação e conceitos do paradigma de Orientação a Objetos facilita o entendimento.`

### 2.1. Descrição, análise e solução inicial 

Na entrevista que eu fiz, me foi pedido para fazer o seguinte:

*Implemente o seguinte:*
```java
        var lb = new LoadBalancer();

        lb.addResource("url1");
        lb.addResource("url2");
        lb.addResource("url3");

        lb.next(); //Retorna a string 'url1'
        lb.next(); //Retorna a string 'url2'
        lb.next(); //Retorna a string 'url3'
        lb.next(); //Retorna a string 'url1' novamente
```

Esse trecho de código me diz que eu preciso ter uma classe LoadBalancer onde tenha um método em que adiciona o recurso (addResource()) e um método em que me retorna a string (next()). Este trecho, numa análise superficial, já me diz o seguinte: 

 1. O principal elemento objetivo dessa classe LoadBalancer é obter uma URL (em forma de string); 
 2. Muito provavelmente, a melhor forma de guardar esses elementos, será numa lista de strings;
 3. A forma como o trecho lb.next() espera o retorno de forma cíclica, ou seja, acessar os elementos em Round-Robin (em termos práticos: acesso circular). Ou seja, partindo do princípio que eu tenho uma lista que armazena essas URLs, cada vez que eu chamo o método next(), ele me retorna o próximo elemento da lista.

Sendo assim, minha primeira ideia de implementação foi a seguinte:

```java
public class LoadBalancer {

    private final List<String> resources;
    private int currentIndex = 0;

    public void addResource(String resource) {
        resources.add(resource);
    }

    public String next() {
        if (resources.isEmpty()) {
            return null;
        }

        String resource = resources.get(currentIndex);
        currentIndex = (currentIndex + 1) % resources.size(); // Este trecho aqui me garante que meu índice para acessar a lista sempre estará no conjunto 
                                                              // [0, listSize - 1] e acessando de forma circular
        return resource;
    }
}
```

### 2.2. Complicando o problema

Ou seja, a implementação dessa forma já atenderia inicialmente. Agora vamos complicar um pouquinho mais. Agora, ao invés de acessar essas URLs de forma circular (round-robin), gostaria de acessar essas URLs de forma aleatória. A partir daí eu tenho uma infinidade de modos de fazer. Uma das formas poderia ser:

```java
public class LoadBalancer {

    private Random random = new Random();

    /*resto dos atributos e métodos*/

    public String next() {
        if (resources.isEmpty())
            return null;

        int randomIndex = this.random.nextInt(resources.size());
        return resources.get(randomIndex);
    }
}
```

Aqui eu substituí a antiga solução pelo método aleatório utilizando a classe Random da própria biblioteca java.util. Agora vamos complicar ainda mais. Adicionarei as seguintes condições:
 * Muitas outras instâncias do problema querem acessar esta classe LoadBalancer e acessar URLs armazenadas nesta mesma lista;
 * Algumas instâncias querem acessar de forma aleatória os elementos da lista e outras querem acessar de forma circular (round-robin);
 * O método next() não pode ser alterado.*

`É fácil explicar porque o método next() não pode ser alterado. Pensando numa situação onde esta classe LoadBalancer é uma biblioteca e tenho códigos espalhados acessando a mesma, se eu fizesse um método diferente que faça algo diferente, eu teria que mudar em todos os lugares e todas as implementações que utilizam esta classe`

Já que eu não quero (e nem posso) mudar a assinatura do método, me vejo "obrigado" a utilizar alguma solução pronta para este problema. E esta solução pronta é algum dos vários padrões de design. E o que eu utilizarei aqui é o padrão "Strategy".

## 3. Utilizando o Padrão Strategy (Behavioral Pattern)

### 3.1 Definição
> "Define uma família de algorítmos, encapsula cada um e torna cada um intercambiável. A estratégia permite que o algoritmo varie independentemente dos clientes que o utilizam."
> 
> Fonte: Livro "Design Patterns - Elements of Reusable Object-Oriented Software" (1994), página 315

O padrão Strategy faz parte dos grupos de padrões comportamentais (Behavioral Patterns). Ele pode ser definido como uma estrutura genérica abaixo:

```mermaid
classDiagram
  class Context {
    + setStrategy(Strategy)
    + executeStrategy(): void
  }

  interface Strategy {
    + algorithmInterface(): void
  }

  class ConcreteStrategyA {
    + algorithmInterface(): void
  }

  class ConcreteStrategyB {
    + algorithmInterface(): void
  }

  Context --|> Strategy
  Strategy o-- ConcreteStrategyA
  Strategy o-- ConcreteStrategyB

```

## Referências:

 * [Livro "Design Patterns - Elements of Reusable Object-Oriented Software" (1994)](https://github.com/TushaarGVS/Design-Patterns-Mentorship/blob/master/Erich%20Gamma%2C%20Richard%20Helm%2C%20Ralph%20Johnson%2C%20John%20M.%20Vlissides-Design%20Patterns_%20Elements%20of%20Reusable%20Object-Oriented%20Software%20%20-Addison-Wesley%20Professional%20(1994).pdf)

 * [¹ Balanceamento de Carga](https://pt.wikipedia.org/wiki/Balanceamento_de_carga)

 * [Round-Robin](https://pt.wikipedia.org/wiki/Round-robin)

 * [Mermaid.io para diagrama de classes](https://mermaid-js.github.io/)
