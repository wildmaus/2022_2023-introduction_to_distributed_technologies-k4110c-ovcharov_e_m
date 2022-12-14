# Why pancake farming contract is brilliant.

### введение
Фермы панкейксвапа позволяют вам получать дополнительный доход в виде токенов cake за стейкинг lp токенов. Но речь в этой статье пойдет не о том как сделать это эффективнее. В данной статье я разберу смарт-контакт фарминга и матетамику стоящую за ним. За время работы разработчиком смарт-контрактов я успел написать разные вариации стейкинга, но глобально их можно разделить на 2 типа. 

Первый - каждый пользователь получает определенный процент награды, начилсяемый либо с учетом реинвестирования, либо без него. Награды чаще всего выплачиваются из определенного пула, а пользователи выбирают время, на которое они стейкают токены (чем дольше тем больший процент они будут получать).

Второй - пользователь получает часть награды пропорционально доле своего стейка от общего числа застейканных токенов. Каждый промежуток времени (1 секунда, 1 блок, 1 день и т.д) происходит распределение определенного числа токенов. Фарминг панкейка относится как раз к второму типу.

### математическая формула расрпеделения
Рассмотрим формулу начисления токенов по узаканным правилам. Для начала нам нужно знать какую часть от общего имеет пользователь.
`userPercent = userStakeAmount / totalStakeAmount`
Получается пользователь при каждом распределении награды должен получать эту часть от распределяемого количества токенов.
`userRewardStepAmount = rewardPerStep * userPercent`
Следовательно на момент вывода пользователь должен получить количество токенов равное сумме распределенных ему наград на каждом шаге распределения
`userRewardAmount = sum(userRewardStepAmount[i])`, где `i` - шаги распределения награды
следовательно раскрывая формулу получаем
`userRewardAmount = sum(rewardPerStep[i] * userStakeAmount[i] / totalStakeAmount[i])`
Таким образом для начисления награды пользователям нужно на каждом этапе распределения проходить по каждому циклом и вычислять их процент на текущий момент. Сответственно если у нас 10 пользователей, то это еще приемлимые цифры, однако с ростом числа пользователей это число будет расти и в итоге только на поддержание системы будут уходить огромные суммы. <!-- переписать последнее предложение, лучше раскрыть такой цикл, возможно посчитать сложность асимптотическую -->

### как панкейк изменил ее
Для исключения цикла и сведения расчета награды к константному времени панкейки изменили формулу для расчета награды.
Рассмотрим последнюю формулу подробнее. Из переменных у нас `totalStakeAmount[i]` т.к. другие пользователи могут вносить и выводить токены и `rewardPerStep`. Будем рассматривать userStakeAmount как константу, потому что при любом ее изменении начисленная награда может быть выведена/сохранена (конец текущего промежутка), а дальше начисления будут проводиться уже на измененный `userStakeAmount` (начало нового промежутка).
Следовательно можно вынести это слагаемое за сумму
`userRewardAmount = userStakeAmount * sum(rewardPerStep[i] / totalStakeAmount[i])`

Вынесем сумму из указанной выше формулы в отдельную переменную - `accCakePerShare`. Она будет сохранять сумму всех `rewardPerStep[i] / totalStakeAmount[i]`, с самого первого шага распределения и до последнего. Тогда между любыми изменениями rewardPerStep и totalStakeAmount это слагаемое будет константой. То есть пусть n cекунд никто не вносит и не выводит токены, а rewardPerStep вообще не изменяется. Тогда за эти n секунд будет начисленно `n * rewardPerStep / totalStakeAmount`. Таким образом за счет умножения 3х слогаемых (константная операция) происходит начисление наградных токенов всем участникам стейкинга! 
Но осталось решить еще одну проблему. Т.к в `accCakePerShare` сохраняются все суммы от начала распределения, то нужно дополнительно исключить часть токенов, в распределении которых пользователь не участвовал. Для этого при изменении стейка пользователя, мы записываем значение `accRewardPerShare` на момент этого события и позже начисляем пользователю только часть равную `userStakeAmount * (accRewardPerShere[end] - accRewardPerShere[start]) = userStakeAmount * sum(rewardPerStep[i] / totalStakeAmount[i])`, где `i = (start : end]`
Вот таким образом панкейки свели начисление награды пользователю до константной сложности! : fire :

### вырезки из реализации
рассмотрим контракт реализации приведенных выше формул (можно нарисовать картиночку тип контракт внутри)

Для начала для начисления наград панкейк вводит пулы - разные пулы для разных токенов. Именно в них хранится accCakePerShere, а также таймстемп последнего обновления. Т.к. контракты не умеют работать с дробными числами для сохранения точности и уменьшения погрешностей целочисленной арифметики accRewardPerShere хранит значение описанной выше суммы умноженное на 10^12. 
*В своих контрактах я использовал значения 10^24, чтобы сохранить больше знаков после запятой (uint256 позволяет хранить значения до 10^76 так что нет опасности переполнения).*
```
struct Pool {
    uint256 accCakePerShare;
    uint256 lastUpdateBlock;
    ...
}
```
Для пользователя также есть структура. В ней сохраняется размер стейка, значение `accRewardPerShere * amount` на момент последнего изменения `amount` пользователя (то есть та часть распределенных токенов, в которой пользователь еще не участвовал). В последней версии панкейк добавил возможность бустинга. Пользователь получает больше награды за счет того, что его amount _виртуально_ увеличивается в несколько раз (в зависимости от значения множителя). Таким образом стейк пользователя составляет большую часть от общего чем без бустинга следовательно пользователь получает больше награды. При любом изменении `amount` пользователя ему отправляется полученное число токенов, а rewardDebt обновляется до этого момента (т.к. за предыдущий период пользователь уже вывел награду).
*В своих контрактах я добавлял в структуру пользователя поле `accReward` для сохранения уже полученной награды пользователем при изменении его `amount`, т.к. в них была предусмотрена блокировка вывода на определенный период.*
```
struct UserInfo {
    uint256 amount;
    uint256 rewardDebt;
    uint256 boostMultiplier;
}
```
Перед каждым изменении размера стейка каким либо пользователем вызывается метод `updatePool`. Он обновляет `accRewardPerShare` добавляя в него `(block.number - lastUpdateBlock) * rewardPerStep / totalStakeAmount`, то есть обновляя сумму до актуального состояния. После чего пользоватю рассчитывается накопленная до изменения стейка награда -  `amount * accRewardPerShere - rewardDebt` (и она сразу отправляется пользователю в случае панкейка). Затем обновляются `rewardDebt` и `totalStakeAmount`, а пользователь получает дальнейшие награды уже на измененный `amount`.

### Схемы автофарминга
Из описанного выше следуюет, что награда начисляется пользователю только на внесенный им размер стейка, то есть в данной схеме отсутствует возомжность расчета награды с учетом реинвестирования. К сожалению это то ограничение с которым приходится мириться для дешевизны расчета награды. Однако есть способы реализовать механизм реинвестирования вручную. Для этого достаточно выводить награду раз в какой-то период (раз в день/неделю/месяц) и стейкать сумму начисленной награды и изначального стейка. Единственное условие - необходимо чтобы суммарное увеличение прибыли компенсировало затраты на реинвестирование.

Для решения этой проблемы были созданы платформы для автофарминга. Эти платформы преставляют собой набор смарт-контрактов и бекенда. Пользователь который хочет получать доход от фарминга с реинвестированием стейкает свои токены не в оригинальный фарминг, а в смарт-контракт этой платформы. Этот контракт собирает все токены пользователей и стейкает их от своего лица в смарт-контракт фарминга. Таким образом смарт-контракт платформы может управлять общим стейком. За механизм реинвестирования отвечает бекенд. Он вызывает методы смарт-контракта платформы, которые выполняют действия, аналогичные ручному реинвестированию, описанному выше. Однако в этот раз вывод и стейк происходит в рамках одной транзакции для большого числа пользователей, что позволяет снизить затраты на реинвестирование в N раз, где N - число участников автофарминга. Однако бекенд платит за операции реинвестирования и, для покрытия расходов, бекенд свапает часть полученной суммарной награды на нативку. Таким образом пользователи получают увеличенную прибыль, т.к. размер их стейка постоянно растет и следовательно они получают большую часть распределяемой награды.

### summary
фарминг панкейка один из популярных паттернов проектирования смарт-контрактов и это не удивительно, учитывая как он свел сложность рассчетов награды в цикле неопределенной длинны к константной!