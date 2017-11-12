## Problem

Dostaliśmy w „spadku” legacy codebase. Takie legacy z piekła rodem. Klasy po 10k linii,
zależności przez '''new''', odwołania do złożonych statycznych metod, pomieszanie logiki domenowej z sql i http.
Widzieliśmy wszyscy nie raz takie twory i nie raz musieliśmy z nimi pracować. Są częścią naszej rzeczywistości, czy to nam się podoba, czy nie.
 
Tak długo, jak nie musimy dotykać legacy wszystko jest dobrze. To sobie jakoś działa, nikt nie wie, jak i dlaczego, ale działa. Zarabia pieniądze.

Przychodzi jednak czas, gdy ktoś chce, abyśmy zmienili coś w logice legacy. Dramat maluje się na twarzy.
Co Teraz? Dołożyć kolejny '''if''' gdzieś na szczycie tego dziesięcio tysiącznika i modlić się że zadziała.

A może przepisać?

Oczywisty problem z legacy jest taki, że nie ma ono testów i jest nie testowalne unitowo. Nie możemy nic refaktoryzować, bo nie mamy testów,
nie mamy testów, bo trzeba by najpierw refaktoryzować kod do 'testowalnej' postaci. Wpadamy w pętlę. Co nam pozostaje? „Refactor and pray"? Refaktoryzować i modlić się, że nic nie popsuliśmy i następnie napisać testy do nowego pięknego kod?

Oczywiście zawsze możemy napisać testy funkcjonalne np.: w codeception które będą testować naszą aplikację przez UI.
Problem z testami UI jest taki, że są:
* Wolne, nie możemy mieć ich zatem zbyt wielu, a co za tym idzie, nie pokryjemy wszystkich przypadków, najwyżej kilka najważniejszych
* Nie zawsze wykonalne. Nie możemy (i nie powinniśmy) w takich testach nic mockować np.: bramki płatności, czasem może być to ograniczeniem.

Co, jednak jeśli jest możliwość przetestowania unitowo legacy kodu? Postaram się pokazać jakie wyzwania czekają przed nami
w testowaniu legacy i jakie mamy w naszym PHP-owym arsenale narzędzia, które mogą pomóc te problemy obejść.

## Uwaga! Poniższe przykłady i narzędzia pokazują jak radzić sobie z testowaniem już istniejącego legacy. Nie oznacza to, że można pisać teraz taki kod, trzymajmy się dobrych praktyk i twórzmy testowalny kod.

### Założenia

Wszystko ma działać w „czystym” PHP. Żadnych extensions, runkitów, tylko zależności PHP-owe

### 1. Wbudowane funkcje php

PHP nie jest językiem w pełni obiektowym. Jeśli mamy w kodzie wywołania natywnych funkcji PHP, nie mamy możliwości sterowania ich działaniem.

Niech będzie sobie klasa Profiler, która ma za zadanie zmierzyć czas wykonania kodu:

```
<?php

namespace PHPUnitAlt;

class Profiler
{
    /**
        * @param callable $callable
        * @return float
        */
       public function profile(callable $callable)
       {
           $start = microtime(true);
   
           $callable();
   
           return microtime(true) - $start;
       }
}

```

Nie możemy sterować tym, co zwraca microtime, nie możemy więc określić co metoda ma zwrócić.
Rozwianie? Można oczywiście wrapować microtime w jakąś klasę helpera i ją wstrzyknąć, ale takie rozwiązanie to:
* Dodatkowy narzut pracy
* W legacy może być niemożliwe (brak DI)

Co więc możemy zrobić? Z pomocą przychodzi [PHPMock](https://github.com/php-mock/php-mock)

Nasz test mógłby wyglądać tak:

```<?php
 
    /**
       * @test
       */
      public function shouldMeasureExecutionTime()
      {
          $time = $this->getFunctionMock((new \ReflectionClass(Profiler::class))->getNamespaceName(), 'microtime');
          $time->expects(static::at(0))->willReturn(1);
          $time->expects(static::at(1))->willReturn(2);
  
          $profiler = new Profiler();
          $result = $profiler->profile(function () {
              // sleep symuluje wykonania jakiegoś kodu, dla celów testowych
              sleep(1);
          });
  
          static::assertEquals(1, $result);
      }
```

Co tu mamy? Mockujemy wbudowaną funkcję microtime. Dzięki temu możemy dokładnie określić, co powinna zwrócić metoda.

Jak to się dzieje? Namespace fallback, gdy wywołujemy microtime, PHP najpierw szuka takiej funkcji w bierzącym namespace.

Jeśli interesuje nas tylko to czy funkcja została wywołana i nie ma potrzeby sterować zwracaną wartością, możemy stworzyć „spy”:

```
   /**
      * @test
      */
     public function shouldCallMethods()
     {
         $spy = new Spy((new \ReflectionClass(Profiler::class))->getNamespaceName(), 'microtime');
         $spy->enable();
 
         $profiler = new Profiler();
 
         $result = $profiler->profile(function(){
             sleep(1);
         });
 
         static::assertEquals(2, count($spy->getInvocations()));
     }
       
 ```

#### Ograniczenia

Nie działa dla funkcji wywołanych z explicit globalnym namespace np.:

```
\microtime()
```

### 2. Metody statyczne

Co jeśli w kodzie mamy wywołania staticów, które np.: strzelającą do bazy? Pomóc nam może [Mockery](http://docs.mockery.io/en/latest/reference/instance_mocking.html?highlight=alias) z funkcjonalnością 'alias'

Niech będzie sobie klasa:

```
<?php
   
   
   namespace TestingLegacy;
   
   
   class StaticExample
   {
       /**
        * @return int
        */
       public function foo()
       {
           $entities = DB::getAll();
   
           // do some logic that we need to test
   
           //
           return count($entities);
       }
   }
```
 
Chcemy przetestować logikę znajdującą się w metodzie, niestety jest ona zależna od zwrotki ze statycznej metody getAll.
Jak może nam tutaj pomóc mockery?

```
<?php


namespace TestingLegacy\Tests;

use PHPUnit\Framework\TestCase;
use TestingLegacy\Entity;
use TestingLegacy\StaticExample;

class StaticExampleTest extends TestCase
{
    /**
     * @test
     */
    public function shouldReturnEntityCount()
    {
        $mock = \Mockery::mock('alias:\TestingLegacy\DB');

        $mock->shouldReceive('getAll')->andReturn([
            new Entity('a'),
            new Entity('b'),
            new Entity('c')
        ]);

        $result = (new StaticExample())->foo();

        static::assertEquals(3, $result);
    }
}
```
#### Ograniczenia

Raz aliasowana metoda nie może zostać zredefiniowana ani przywrócona do pierwotnej postaci w tym samym procesie. Jeśli mamy x testów,
gdzie każdy test mockuje ta samą metodę, ale w inny sposób (np.: ma zwracać inna wartość) to musimy uruchomić testy w osobnych procesach, co negatywnie odbije się na czasie wykonania suitu testów jednostkowych.

### 3. I don't want do die()

Co, jeśli w kodzie mamy np.: die()?
Pomóc może [Patchwork](http://patchwork2.org/) - biblioteka do monkey patchingu.
Pozwala nie tylko na redefinicję metod statycznych, ale także metod prywatnych i wbudowanych funkcji oraz konstruktów PHP.

Przykładowa klasa testowana:

```
<?php


namespace TestingLegacy;


class LegacyClass
{
    /**
     * @param array $data
     * @return int
     */
    public function process(array $data)
    {
        if(empty($data)){
            die();
        }

        $repository = new Repository();

        $entities = $repository->getAll();
        if(empty($entities)){
            echo 'error';
        }

        // do some logic
        return 1;
    }

   
    
}
```

Chcemy sprawdzić np.: czy pierwszy guard zadział i przerwał działanie metody. Niestety die() przerwie też wykonanie naszego testu.
Jak poradzić sobie z tym korzystając z patchwork? Monkey patching pozwala nam redefiniować dowolną linię w kodzie:

```
/**
     * @test
     */
    public function shouldReturnEarly()
    {
        $this->expectException(\Exception::class);
        $this->expectExceptionMessage('I died');

        $sut = new LegacyClass();

        redefine('die', function () {
            throw new \Exception('I died');
        });

        $result = $sut->process([]);
    }
```

Podmieniamy w kodzie die na exception, który z kolej możemy przechwycić w teście i sprawdzić jego typ oraz wiadomość.

#### Ograniczenia

* Nadpisywane funkcje / metody muszą być z góry określone w pliku patchwork.json
* Biblioteka ma pewne ograniczenia, nie można nadpisać np.: statycznych metod PDO, nie można też nadpisać ''new'' (nad tym ostatnim trwają prace)
* Problemy wydajnosciowe.

### 4. Broń ostateczna.

Gdy mamy do czynienia z paskudnym legacy gdzie pełno jest staticów, new, die, exit oraz innych rzeczy skutecznie uniemożliwiających nam testowanie jednostkowe, pomóc może tylko [Kahlan](https://github.com/kahlan/kahlan)

W przeciwieństwie do ww. Kahlan jest osobnym frameworkiem, tak więc nie możemy go użyć z poziomu PHPUnit-a. Kahlan, podobnie jak patchwork, wykorzystuje monkey patching.
Zobaczymy jak to działa w praktyce. Wrócmy do naszej legacy klasy:

```
class LegacyClass
{
    /**
     * @param array $data
     * @return int
     */
    public function process(array $data)
    {
        if (empty($data)) {
            die();
        }

        $repository = new Repository();

        $entities = $repository->getAll();
        if (empty($entities)) {
            echo 'error';
        }

        // do some logic
        return 1;
    }


}
```

Kahlan oferuje nam tzw. describe-it syntax. Oprócz standardowych dla frameworka testowego funkcjonalności takich jak assercje i stuby/mocki, mamy też możliwość monkey patchingu (bez potrzeby definiowania patchowanych funkcji jak w przypadku patchwork)
Przykładowy test:
```
<?php

use Kahlan\QuitException;
use Kahlan\Plugin\Quit;

describe('Legacy class', function () {
    describe('process method', function () {
        it('should return early', function () {

            Quit::disable();

            $sut = new \TestingLegacy\LegacyClass();

            $closure = function () use ($sut) {
                $sut->process([]);
            };
            expect($closure)->toThrow(new QuitException('Exit statement occurred'));

            Quit::enable();
        });

        it('should echo error when no entities found', function () {

            $repositoryMock = \Kahlan\Plugin\Double::instance(['class' => \TestingLegacy\Repository::class]);
            allow($repositoryMock)->toReceive('getAll')->andReturn([]);
            allow(\TestingLegacy\Repository::class)->toBe($repositoryMock);
            $sut = new \TestingLegacy\LegacyClass();

            $closure = function () use ($sut) {
                $sut->process(['same_data']);
            };
            expect($closure)->toEcho('error');
        });
    });
});
```

Obiekt Quit pozwala na nadspisywanie die() i exit(), zamiast skończyć skrypt, poleci wyjątek.
Możemy też m.in. przechwycić 'echo'. Zwróćmy uwagę też, że w testowanym kodzie było 'new', Kahlan pozwala nam na wstawienie mocka
bezpośrednio w kod za pomocą monkey patching. Kahlan ma praktycznie nie ograniczone możliwości w testowaniu kodu PHP. Co więcej, nie musi
służyć tylko i wyłącznie jako narządzie do testowani legacy. Można go z powodzeniem wykorzystać jako alternatywę dla PHPUnit.

#### Ograniczenia
* Nowy framework, niekompatybilny z PHPUnit. Wymaga czasu na jego nauczenie się.

### Słowo na koniec

Mamy do dyspozycji narzędzia. Czy i kiedy powinniśmy ich używać? Jak zwykle „to zależy". Zależy od naszego projektu, naszej mocy przerobowej i naszych potrzeb. Należy pamietać, że narzędzie te mają swoje ograniczenia i należy ich używać rozsądnie.

Jako że w tytule bloga są suchary, tak więc na zakończenie:

What's the second movie about a database engineer called?

The SQL.

### Kod

Kod dostępny pod https://github.com/mieszkomalawski/legacy-unit












