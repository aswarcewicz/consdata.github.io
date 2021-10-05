---
layout:     post
title:      "Zarządzanie stanem aplikacji frontendowej na przykładzie NgRx"
date:       2021-09-21 6:00:00 +0100
published:  true
didyouknow: false
lang:       pl
author:     mchrapkowicz
image:      /assets/img/posts/2021-09-31-zarzadzanie-stanem-aplikacji-frontendowej/ngrx.webp
description: "W miarę rozwijania złożonych aplikacji webowych ważnym i nieoczywistym zagadnieniem staje się projektowanie przepływu informacji pomiędzy komponentami. W tym artykule poznamy jedną z implementacji architektury Redux wykorzystującej koncepcję globalnego stanu aplikacji oraz pokażemy przykład jej zastosowania w bibliotece NgRx."
tags:
- frontend
- redux
- state management
- zarządzanie stanem
- store
- reducer
- action
- effect
- angular
- react
---

W miarę rozwijania złożonych aplikacji webowych ważnym i nieoczywistym zagadnieniem staje się projektowanie przepływu informacji pomiędzy komponentami. Często mamy do czynienia z wieloma źródłami danych. Mogą to być na przykład najróżniejsze zewnętrzne serwisy czy polecenia wprowadzane przez użytkownika. Podstawowym podejściem jest przekazywanie tych danych przy pomocy rozwiązań dostarczanych przez wykorzystywany framework (na przykład data binding w Angularze), jednak bardzo łatwo można wyobrazić sobie sytuację, w której informacje, z których chcemy skorzystać, znajdują się w zupełnie innej części aplikacji, a przed nami stoi zadanie „przepchania" ich aż do interesującego nas miejsca. Jest to oczywiście rozwiązanie nieskalowalne, trudne w utrzymaniu i niezbyt eleganckie.

Wyobraźmy sobie zatem sytuację, w której wszystkie możliwe dane w sposób uporządkowany trafiają do jednego miejsca i mogą być z niego odczytane w dowolnej chwili. Prawda, że cudna koncepcja? Nazywamy ją "single source of truth", czyli, w wolnym tłumaczeniu, jedno źródło prawdy.  Wykorzystanie takiego pomysłu pozwala pisać bardziej przewidywalny i łatwiej testowalny kod, jak również znacząco zwiększa możliwość skalowania aplikacji. W tym wpisie postaram się nieco przybliżyć jedną z implementacji architektury Redux wykorzystującej koncepcję globalnego stanu aplikacji oraz pokażę przykład jej zastosowania w bibliotece NgRx.

### Czy na pewno potrzebujesz zarządzania stanem aplikacji?
Zanim zaczniemy, chciałbym jeszcze zwrócić uwagę, że nie każda aplikacja jest tak rozbudowana i złożona, żeby potrzebować całego mechanizmu zarządzania stanem. W ustaleniu, czy tak jest, pomaga zasada SHARI zaprezentowana, chociażby, w [oficjalnej dokumentacji NgRx](https://ngrx.io/guide/store/why). Warto sobie odpowiedzieć, czy potrzebujemy stanu, który jest:
- **S**hared - współdzielony pomiędzy wiele komponentów i serwisów.
- **H**ydrated - trwały i z możliwością ponownego zasilenia z zewnętrznego źródła jak np. local storage.
- **A**vailable - dostępny cały czas, niezależnie od sposobu nawigacji po aplikacji, np. podczas przechodzenia i cofania się w aplikacji prezentującej złożony wniosek.
- **R**etrieved - zdolny do przechowywania danych pochodzących z zewnętrznych źródeł, co pozwala na przykład zapisać wynik żądania HTTP i nie wykonywać go po raz kolejny w celu otrzymania tych samych informacji.
- **I**mpacted - zdolny do zmieniania się pod wpływem akcji wykonywanych przez komponenty i serwisy.

## Architektura Redux
No dobrze, to jeśli już wiemy, że w naszym dobrym interesie leży skorzystanie z dobrodziejstw architektury Redux, to spójrzmy, co tak naprawdę przez to rozumiemy.

![Redux](/assets/img/posts/2021-09-31-zarzadzanie-stanem-aplikacji-frontendowej/redux.png)
<span class="img-legend">Architektura Redux z lotu ptaka</span>

Na razie powyższy diagram może wydawać się nieco tajemniczy, zatem już spieszę z wyjaśnieniami.

* Jak wspomniałem we wstępie, Redux zakłada istnienie globalnego, niemutowalnego bytu, przechowującego stan aplikacji - **store**. Fizycznie jest to obiekt w formacie JSON, którego struktura definiowana jest przez programistę. 
* W celu wywołania zmian w store komponenty aplikacji muszą wywoływać **akcje** (ang. _actions_). Reprezentują one konkretne zdarzenia zachodzące w systemie i niosą ze sobą konkretne informacje.
* Akcje są przechwytywane przez **reducery** (ang. _reducers_). Reducerem nazywamy czystą funkcję (ang. _pure function_), która konsumuje akcję i, w zależności od jej przeznaczenia oraz zgodnie z logiką reducera, zwraca odpowiednio zmodyfikowany, nowy store. 
* Dane ze store do komponentów trafiają poprzez **selektory** (ang. _selectors_). Wszystkie zmiany stanu aplikacji są propagowane do reagujących na nie selektorów dzięki mechanizmowi Observable znanym z biblioteki RxJs, o której więcej można przeczytać [w tym wpisie]({% post_url pl/2020-01-09-rxjs-introduction %}). Selektory powinny otrzymywać wyłącznie dane potrzebne do działania komponentu, w których są używane.
* W przypadkach, kiedy wywołanie akcji powinno pociągnąć za sobą dowolne działanie niezwiązane bezpośrednio z aplikacją (np. zapytanie do bazy danych, czy żądanie do zewnętrznej usługi) - do gry wchodzą **efekty** (ang. _effects_). Podobnie jak reducery potrafią one reagować na konkretne akcje i wykonać przypisane im zadanie. Mogą one również wywołać kolejną akcję, która trafi do reducera, a co za tym idzie - do store. Wywołania efektów mogą być zarówno synchroniczne, jak i asynchroniczne.

## Implementacja Redux w NgRx
Przy pierwszym kontakcie wszystkie powyższe pojęcia i pomysły mogą wydawać się nieco skomplikowane i nadmiarowe, zatem na prostym przykładzie pokażę, jak wyglądają fragmenty aplikacji pisanej przy pomocy NgRx. Na końcu wpisu znajduje się link do repozytorium, w którym widać, jak wygląda cała aplikacja oraz jej konfiguracja. Również z tego powodu pomijam instrukcje instalacji biblioteki i konfiguracji środowiska.

Załóżmy, że chcemy stworzyć prosty program pozwalający na zapisywanie sposobów na pokonanie nudy. Jednym z możliwych do podjęcia działań będzie wywołanie API, które na każde żądanie odpowie pomysłem na jakąś aktywność. Drugą opcją będzie dodanie własnej koncepcji poprzez prosty formularz. Zacznijmy zatem.

### Store
Na samym początku procesu, warto uzmysłowić sobie, co tak naprawdę chcemy przechowywać w naszym store, a potem powiedzieć to Typescriptowi tworząc interfejs. W naszym przypadku będziemy przechowywali w nim jedynie listę czynności potencjalnie zabijających nudę, jednak nawet w tak prostym przypadku warto zwrócić uwagę na to, jaką strukturę planujemy nadać store'owi. Początkowo najprościej jest operować na płaskiej strukturze, w której wszystkie dane znajdują się na tym samym poziomie.
Niestety przy rozwijaniu aplikacji takie podejście staje się bardzo nieefektywne, zarówno pod względem organizacji, jak i, z czasem, również wydajności. Zdecydowanie lepiej jest podzielić store na tak zwane "slice'y", czyli wycinki danych, na przykład dzieląc store według funkcjonalności aplikacji. Zobaczmy takie podejście na przykładzie:

```typescript
export interface State {
    activitiesState: ActivityItemsState;
}

export interface ActivityItemsState {
    activities: ActivityItemModel[];
}

export interface ActivityItemModel {
    name: string;
    participants: number;
}

```

Przejdźmy po powyższych interfejsach po kolei. Pierwszy z nich mówi, że spodziewamy się mieć jeden wycinek danych (slice) typu `ActivityItemsState`, w którym będą przechowywane wszystkie informacje dotyczące aktywności dodanych do aplikacji. Następnie definiujemy, co tak naprawdę znajduje się w tym wycinku danych - jest to tablica obiektów typu `ActivityItemModel`, czyli informacji o różnych aktywnościach. Ostatni interfejs to już wyłącznie definicja modelu biznesowego - w naszej aplikacji będziemy mieli do czynienia z nazwą czynności oraz możliwą liczbą jej uczestników.

Taki podział store zapewnia bardzo łatwą jego rozszerzalność, gdyż w przypadku dodania nowej funkcjonalności wystarczy dopisać nowy wycinek danych do store. W bardzo dużych aplikacjach, gdzie obiekt stanu aplikacji jest już dość potężny, to podejście ma jeszcze jedną zaletę - pozwala wykorzystać _lazy loading_, to znaczy wczytywać tylko konkretne wycinki store, potrzebne do aktualnie przetwarzanej części aplikacji. 

### Akcje
Potrzebujemy teraz możliwości wywoływania akcji, żeby powodować zmiany w store. Aplikacja będzie przewidywała dwa zdarzenia: wystosowanie żądania do API po pomysł na jakieś zajęcie oraz dodanie aktywności do listy. Wygląda to następująco:

```typescript
export const addActivityType = 'Add activity';
export const getActivityType = 'Get activity';

export const addActivity = createAction(addActivityType, props<ActivityItemModel>());
export const getActivity = createAction(getActivityType);
```

Każda akcja musi mieć zdefiniowany swój typ. NgRx używa do tego tzw. _literal type_, czyli po prostu używa typu string do identyfikacji akcji (u nas odpowiednio `Add activity` oraz `Get activity`. W nazywaniu akcji mamy pełną dowolność. Dodatkowo w przypadku dodawania nowej aktywności do listy musimy ją przekazać w ciele akcji przy pomocy metody `props<>()`. Dzięki wykorzystaniu funkcji `createAction` z biblioteki NgRx tworzenie akcji, jak widać, jest bardzo proste.

### Efekt
Jak wspomniałem, będziemy korzystać z zewnętrznego API. W tym celu napiszemy efekt, który będzie reagował na akcję `Get activity`, następnie wystosowywał żądanie HTTP, a po otrzymaniu odpowiedzi wywoływał akcję `Add Activity`.

```typescript
@Injectable()
export class ActivityEffect {
    constructor(private actions$: Actions,
                private http: HttpClient) {
    }

  getActivity$ = createEffect(() => this.actions$.pipe(
    ofType(getActivityType),
    mergeMap(() => this.http.get<ActivityItemModel>('https://www.boredapi.com/api/activity')
      .pipe(
        map(response => (addActivity(response))),
        catchError(() => EMPTY)
      ))
  ));
}
```

Pojawia się tutaj dość dużo zagadnień, zatem spójrzmy na ten fragment kodu od góry do dołu. Najpierw informujemy Angulara, że efekt to tak naprawdę wstrzykiwalna klasa, podobnie jak zwykłe serwisy. W konstruktorze zapewniamy dostęp do serwisu HTTP oraz do zmiennej typu `Actions`. Jest to w gruncie rzeczy serwis udostępniający wszystkie wywołane akcje od momentu ostatniego uruchomienia reducera (czyli dzięki temu damy radę reagować na przychodzące akcje). 

Następnie definiujemy sam efekt:
* zaglądamy do strumienia akcji 
```typescript 
this.actions$.pipe()
```
* szukamy tylko akcji określonego typu 
```typescript
ofType(getActivityType)
```
* reagujemy na każdą z nich poprzez wywołanie żądania HTTP i zaglądamy do strumienia zawierającego odpowiedź z API 
```typescript 
mergeMap(() => this.http.get<ActivityItemModel>('https://www.boredapi.com/api/activity').pipe()
```
* mapujemy odpowiedź na wywołanie akcji `map(response => (addActivity(response)))`
```typescript 
map(response => (addActivity(response)))
```
* w przypadku błędu zwracamy pustą wartość Obervable
```typescript 
catchError(() => EMPTY)
```

W wyniku działania tego efektu każde wywołanie akcji `Get Activity` przy poprawnej odpowiedzi z API wywoła akcję `Add Activity` z otrzymaną czynnością.

### Reducer
W reducerach zazwyczaj znajduje się najwięcej logiki ze wszystkich komponentów NgRx. To tutaj trzeba zdecydować jak dodawać do store kolejne dane. Warto pamiętać, że reducery to **czyste funkcje**, które przyjmują jako parametr akcję i obecny stan aplikacji, a następnie zwracają nowy stan, dzięki czemu zachowujemy jego niemutowalność. W naszym przypadku wygląda to tak:

```typescript
const initialState: ActivityItemsState = {activities: []};

export const activityReducer = createReducer(
    initialState,
    on(addActivity, (state, payload) => ({
        ...state,
        activities: state.activities.concat({activity: payload.activity, participants: payload.participants} as ActivityItemModel)
    }))
);
```

W pierwszej linii definiujemy jakiego stanu początkowego store (lub - jak ma to miejsce powyżej - jego wycinka) się spodziewamy. W naszym przypadku będzie to pusta tablica. Następnie wywołujemy funkcję `createReducer`, która w pierwszym parametrze przyjmuje wspomniany stan początkowy, a w drugim (i każdym kolejnym) - definicję odpowiedzi na przybyłą akcję. W tym przypadku reducer reaguje na akcję `Add Activity` poprzez dodanie do dotychczasowej tablicy `activities` nowego elementu i zwrócenie całości jako wynik wywołania funkcji anonimowej. Reducer jest częścią całej architektury, którą można w bardzo łatwy sposób przetestować i zapewnić sobie w ten sposób pewność jego poprawnego działania.

### Selektor
Ostatnią częścią układanki w przepływie są selektory. Podobnie jak reducery są one czystymi funkcjami, których zadaniem jest obserwowanie wycinków store i dostarczanie informacji o ich zmianach do komponentów. W naszej aplikacji selektor zdefiniowany jest następująco:

```typescript
const getActivitiesFeatureState = createFeatureSelector<ActivityItemsState>('activitiesState');

export const getActivities = createSelector(getActivitiesFeatureState, state => state.activities);
```

Najpierw, przy pomocy funkcji `createFeatureSelector` tworzymy tzw. feature selector pozwalający na wyciągnięcie pojedynczego wycinka (_slice_) danych, nazwanego przez nas `activitiesState`. Następnie używamy go tworząc właściwy selektor przy pomocy funkcji `createSelector` i wskazując go w pierwszym jej argumencie. Drugim parametrem jest funkcja anonimowa wskazująca, które dane chcemy otrzymać w wyniku działania selektora. W naszym przypadku jest to tablica czynności, o nazwie `activities`. Tak zbudowany selektor jest gotowy do użycia w komponencie.

### Spięcie całości
Mamy już wszystkie potrzebne części logiki NgRx, z których chcemy skorzystać. Pozostaje już tylko dowiedzieć się, jak ich użyć. Pokażę sytuację, w której użytkownik klika przycisk odpowiadający za pobranie przykładowej aktywności z API, przechodząc jednocześnie przez odpowiednie miejsca w kodzie. W trakcie czytania zachęcam do spoglądania na zamieszczony wcześniej w poście diagram przepływu.

1. Zacznijmy od komponentu wywołującego początkową akcję typu `Get Activity`. Budowa klasy tego komponentu może wyglądać następująco (pomijam _template_):
```typescript
export class ActivityApiComponent {

    constructor(private store: Store<State>) {
    }

    getActivity(): void {
        this.store.dispatch(getActivity());
    }
}
```
Jak widać w konstruktorze, wstrzykujemy obiekt Store, na którym wykonujemy metodę `dispatch` podając w jej argumencie typ akcji. Na tym kończy się odpowiedzialność komponentu.

2. Wysłaną akcję przechwytuje efekt, który, zgodnie z kodem zaprezentowanym wcześniej, wysyła żądanie do API, a otrzymawszy odpowiedź przekazuje ją jako argument nowej akcji `Add Activity`.
3. Akcja typu `Add Activity` jest przechwytywana przez reducer, który wyłuskuje z niej przekazaną z API odpowiedź i dodaje do store.
4. W ostatniej kolejności do gry wchodzi selektor, który wykrywa zmianę store wynikającą z działania reducera. Użycie selektora w komponencie może wyglądać następująco:

```typescript
export class ActivityListComponent implements OnInit {

  activities: ActivityItemModel[] = [];

  constructor(private store: Store<State>) {
  }

  ngOnInit(): void {
    this.store.select(getActivities)
      .subscribe(activities => this.activities = activities);
  }

}
```
W metodzie `ngOnInit` wskazujemy, że chcemy reagować na zmiany store przy pomocy selektora `getActivities`, a następnie przy każdej takiej zmianie aktualizujemy lokalnie utrzymywaną tablicę, którą można następnie wykorzystać w ciele komponentu.

Tym samym cały proces dobiega końca, a kolejne kliknięcie przycisku wywoła go od nowa.

## Inne podejścia
Warto wspomnieć, że tak, jak dla Angulara istnieje biblioteka NgRx, tak również dla pozostałych frameworków z tzw. wielkiej trójcy znajdziemy implementacje architektury Redux:
- Redux dla Reacta
- Vuex dla Vue

Należy również zaznaczyć, że Redux nie jest jedynym sposobem na zarządzanie stanem. Można tutaj wymienić takie alternatywy, jak:
* [MobX](https://mobx.js.org/README.html) 
* [Cycle.js](https://cycle.js.org/)
* [Flux](https://github.com/facebook/flux) - warto wiedzieć, że Flux był protoplastą Reduxa
* [Apollo](https://www.apollographql.com/)

## Podsumowanie
Łatwo zauważyć, że na pierwszy ogień NgRx, czy szerzej - Redux - potrafią nieco przytłoczyć ilością kodu, którą trzeba napisać, by nawet drobne funkcjonalności działały. Jest to jeden z największych zarzutów wobec tego rozwiązania, więc jeśli takie były Twoje odczucia podczas czytania tego artykułu - gratuluję krytycznego myślenia! Zauważmy jednak, że po przebrnięciu przez początkowe trudności zostajemy z aplikacją, którą bardzo łatwo możemy otestować, rozszerzać i której działanie jest jasno zdefiniowane.

Mam nadzieję, że tym artykułem zainspirowałem nieco do zainteresowania się tematem zarządzania stanem aplikacji frontendowych. Serdecznie zapraszam do rozpoznawania tematu we własnym zakresie, gdyż przedstawiłem tu zaledwie namiastkę możliwości, jakie zapewnia Redux i NgRx.

### Linki
- [Repozytorium z projektem](https://github.com/Chrrapek/BoredNgRx)
- [Dokumentacja NgRx](https://ngrx.io/docs)