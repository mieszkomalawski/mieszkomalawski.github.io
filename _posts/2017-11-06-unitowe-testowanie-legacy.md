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


