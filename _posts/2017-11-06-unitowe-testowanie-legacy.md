##Problem

Dostaliśmy w "spadku" legacy codebase. Takie legacy z piekła rodem. Klasy po 10k linii,
zależności przez ```new```, odwołania do złozonych statycznych metod, pomieszanie logiki domenowej z sql i http.
Widzieliśmy wszyscy nie raz takie twory i nie raz musieliśmy z nimi pracować. Są częścią naszej rzeczywistości, czy to nam sie podoba czy nie.
 
Tak długo jak nie musimy dotykać legacy wszystko jest dobrze. To sobie jakoś działa, nikt nie wie jak i dlaczego ale działa, zarabia pieniądze.

Przychodzi jednak czas gdy ktoś chce abyśmy zmienili coś w logice legacy. Dramat maluje sie natwarzy. 
Co Teraz? Dołozyć kolejnego ```ifa``` gdzieś na szczycie tego dziesięcio tysięcznika i modlić się ze zadziała. 

A może przepisać? 

Oczywisty problem z legacy jest taki że nie ma ono testów i jest nie testowalne unitowo. Nie możemy nic refaktoryzować bo nie mamy testów,
nie mamy testów bo trzeba by najpierw zrefaktoryzować kod do 'testowalnej' postaci. Wpadmy w pętlę. Co nam pozostaje? "Refactor and pray"? Refaktoryzować i modlić się że nic nie popsuliśmy i następnie napisać testy do nowego pięknego kod?

Oczywiście zawsze możemy napisać testy funkcjonalne np: w codeception które bedą testować nasza aplikację przez UI.
Problem z testami UI jest taki że są:
* Wolne, nie możemy mieć ich zatem zbyt wielu, a co za tym idzie nie pokryjemy wszystkich przypadków, najwyżej kilka najważniejszych
* Nie zawsze wykonalne. Nie możemy ( i nie powinniśmy ) w takich testach nic mockować np: bramki płatności, czasem może być to ograniczeniem
 
Co jednak jeśli jest możliwość przetestowania unitowo legacy kodu? Postaram sie pokazać jakie wyzwania czekają przed nami
w testowaniu legacy i jakie mamy w naszym PHP-owym arsenale narzędzia które mogą pomóc te problemy obejść.

##Uwaga! Poniższe przykłady i narzędzia pokazują jak radzić sobie z testowaniem już istniejącego legacy. Nie oznacza to że można pisać teraz taki kod, trzymajmy się dobrych praktyk i twórzmy testowalny kod.

###Założenia

Wszystko ma działać w "czystym" PHP. Żadnych extensions, runkitów, tylko zależności PHP-owe

### 1. Wbudowane funkcje php

PHP nie jest językiem w pełni obiektowym. Jeśli mamy w kodzie wywyołania natywnych funkcji PHP, nie mamy możliwości sterowania ich działaniem.

Niech będzie sobie klasa Profiler która ma za zadanie zmierzyć czas wykonanania kodu:

```
<?php

namespace PHPUnitAlt;

class Profiler
{
    /**
     * @param callable $callable
     * @return int
     */
    public function profile(callable $callable) : float
    {
        $start = microtime(true);

        $callable();

        $duration = microtime(true) - $start;

        return $duration;
    }
}

```

Nie możemy sterować tym co zwraca microtime, nie możemy więc okreslić co metoda ma zwrócić.
Rozwiąnie? Można oczywiście wrapować microtime w jakaś klasę helpera i ją wstrzyknać, ale takie rozwiązanie to:
* Dodatkowy narzut pracy
* W legacy może być nie możliwe (brak DI)

Co więc możemy zrobić? Z pomoca przychodzi [PHPMock](https://github.com/php-mock/php-mock)

Nasz test mógłby wyglądać tak:

```<?php

/**
     * @test
     */
    public function shouldMeasureExecutionTime()
    {
        $profiler = new Profiler();
        $result = $profiler->profile(function(){
            sleep(1);
        });

        $time = $this->getFunctionMock('PHPUnitAlt', 'microtime');
        $time->expects(static::at(0))->willReturn(1);
        $time->expects(static::at(1))->willReturn(2);


        $result = $profiler->profile(function(){
            sleep(1);
        });

        static::assertEquals(1, $result);
    }

```

Co tu mamy? Mockujemy wbudowaną funkcję microtime. Dzięki temu możemy dokładnie określić co powinna zwrócić metoda.

Jak to się dzieje? Namespace fallback, gdy wywołujemy microtime, PHP najpierw szuka takiej funkcji w bierzącym namespace.

Jeśli interesuje nas tylko to czy funkcja została wywołana i nie ma potrzeby sterować zwracaną wartością, możemy stworzyć "spy":

``` /**
        * @test
        */
       public function shouldCallMethods()
       {
           $spy = new Spy('PHPUnitAlt', 'microtime');
           $spy->enable();
   
           $profiler = new Profiler();
   
           $result = $profiler->profile(function(){
               sleep(1);
           });
   
           static::assertEquals(2, count($spy->getInvocations()));
       }
       
 ```

####Ograniczenia

Nie działa dla funkcji wywyłonych z explicit globalnym namespace np:

```\microtime()```


