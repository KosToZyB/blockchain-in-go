Написание blockchain менее чем за 200 строк кода на Go
===
Представляю вашему вниманию перевод статьи "[Code your own blockchain in less than 200 lines of Go!](https://medium.com/@mycoralhealth/code-your-own-blockchain-in-less-than-200-lines-of-go-e296282bcffc)".

<img src="https://habrastorage.org/webt/ui/te/cq/uitecq3ytvbahfprb2zsnofulpo.jpeg" alt="image" align = "center"/>
Данный урок является хорошо адаптированным [постом](https://medium.com/@lhartikk/a-blockchain-in-200-lines-of-code-963cc1cc0e54) про простое написание blockchain на Javascript.  Мы портировали его на Go и добавили дополнительных фич, таких как просмотр цепочек в браузере.
<cut />

Примеры в уроке будут основываться на данных сердцебиения. Мы ведь медицинская компания. Для интереса, вы можете подсчитать свой [пульс](https://www.webmd.com/heart-disease/heart-failure/watching-rate-monitor#1) (кол-во ударов в минуту) и учитывать это число во время учебного курса.

Почти каждый разработчик в мире слышал про blockchain, но большинство до сих пор не знают, как это работает. Многие слышали только про биткоин, [смарт-контракты](https://ru.wikipedia.org/wiki/%D0%A1%D0%BC%D0%B0%D1%80%D1%82-%D0%BA%D0%BE%D0%BD%D1%82%D1%80%D0%B0%D0%BA%D1%82). Данный пост является попыткой развеять слухи о blockchain, помогая Вам написать свой собственный blockchain на Go менее чем в 200 строк кода! В конце данного урока Вы сможете запустить и записать данные в blockchain локально, а так же просмотреть это в браузере.

Есть ли более хороший способ узнать о blockchain, чем создать свой собственный?

Что вы сможете сделать
---

* Создать свой собственный blockchain
* Понять, как работает хэширование в сохранение целостности цепочки блоков
* Увидеть, как добавляются новые блоки
* Увидите, как разрешаются коллизии, когда несколько узлов генерируют блоки
* Создадите просмотр вашего blockchain в браузере
* Добавите новые блоки
* Получите базовые знания о blockchain

Что вы не сможете сделать
---
Что бы этот пост оставался простым, мы не будем рассматривать более совершенные концепции [proof of work](https://habrahabr.ru/post/263769/) и [proof of stake](https://habrahabr.ru/post/265561/). Сетевое взаимодействие будет моделироваться, что бы Вы могли просматривать Ваш blockchain и просматривать добавленные блоки. Сетевая работа будет зарезервированная для будущих постов.

Давайте начнем!
===
Установка
----
Поскольку мы собираемся писать код на Go, мы предполагаем, что у вас уже есть опыт разработки на нем. После [установки](https://golang.org/dl/) мы так же будем использовать следующие пакеты:
```
go get github.com/davecgh/go-spew/spew
```
Spew позволяет нам красиво выводить структуры и слайсы в консоль.
```
go get github.com/gorilla/mux
```
Gorilla/mux это популярный пакет для написания обработчиков запросов.
```
go get github.com/joho/godotenv
```
```Gotdotenv``` позволяет нам читать из файла ```.env``` который лежит в корне каталога, поэтому нам не придется задавать в нашем коде такие параметры, как http порт.

Давайте создадим наш ```.env``` файл в корне каталога, который будет определять порт на котором мы будем слушать HTTP запросы. Просто добавьте строку в файл:
```
ADDR=8080
```
Создайте файл ```main.go```. Вся реализация будет в этом файле и будет содержать менее 200 строк кода.

Импорты
---
Импорты пакетов, вместе с объявлением пакета:
```
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"log"
	"net/http"
	"os"
	"time"

	"github.com/davecgh/go-spew/spew"
	"github.com/gorilla/mux"
	"github.com/joho/godotenv"
)
```

Модель данных
---
Давайте определим структуру каждого из наших блоков, которые представляют собой blockchain. Чуть ниже мы объясним для чего необходимы все эти поля:
```
type Block struct {
	Index     int
	Timestamp string
	BPM       int
	Hash      string
	PrevHash  string
}
```
Каждый блок содержит данные, которые будут записаны в blockchain и представляет собой событие каждого замера пульса.
* ```Index``` - индекс записи данных в blockchain
* ```Timestamp``` - временная метка, когда данные записываются
* ```BPM``` - удары в минуту. Это частота вашего пульса
* ```Hash``` - идентификатор SHA256, идентифицирующий текущую запись
* ```PrevHash``` - идентификатор SHA256, идентифицирующий предыдущую запись в цепочке

Давайте объявим наш blockchain, который представляет собой просто слайс структур:
```
var Blockchain []Block
```

Итак, как хеширование используется в блоках и в blockchain? Мы используем хэши для определения и сохранения блоков в правильном порядке.  Благодаря тому, что поле ```PrevHash``` в каждом блоке ссылается на поле ```Hash``` в предыдущем блоке (т.е. они равны), мы знаем правильный порядок блоков.
<img src="https://habrastorage.org/webt/uh/qq/9p/uhqq9pcc817-yf9xsusbajckcjc.png" alt="image"/>
</br>
##Хэширование и создание новых блоков
---
Зачем нам хэшировать? Мы получаем хэш по двум основным причинам:
* Чтобы сэкономить место. Хэши производятся из всех данных, находящихся в блоке. В нашем случае есть только несколько блоков данных, но представьте, что у нас есть данные из сотен, тысяч или миллионов предыдущих записей. Намного эффективнее хэшировать эти данные в одну строку SHA256 и хэшировать хеши, чем копировать все данные предыдущих блоков снова и снова.
* Сохранение целостности цепочки. Сохраняя предыдущие хэши, как мы делаем на диаграмме выше, мы можем гарантировать, что блоки в blockchain находятся в правильном порядке. Если злоумышленник захочет присоединитьсяФ и манипулировать данными (например, изменить сердечный ритм, что бы исправить цены на страхование жизни), хэши начнут изменяться и все будут знать, что цепочка "сломана" и все будут знать, что доверять этой цепочки нельзя.

Давайте напишем функцию, которая возьмет наши данные ```Block``` и создаст для них хэш SHA256.
```
func calculateHash(block Block) string {
	record := string(block.Index) + block.Timestamp + string(block.BPM) + block.PrevHash
	h := sha256.New()
	h.Write([]byte(record))
	hashed := h.Sum(nil)
	return hex.EncodeToString(hashed)
}
```
Функция ```calculateHash```  объединяет в одну строку ```Index```, ```Timestamp```, ```BPM```, ```PrevHash``` из структуры ```Block```, которая является аргументом функции и возвращается все в виде строкового представления хэша SHA256. Теперь мы можем сгенерировать новый блок со всеми необходимыми элементами с помощью новой функции ```generateBlock```. Для этого нам нужно будет передать предыдущий блок, что бы мы могли получить его хэш и индекс, а так же передадим новое значение частоты пульса ```BPM```.
```
func generateBlock(oldBlock Block, BPM int) (Block, error) {

	var newBlock Block

	t := time.Now()

	newBlock.Index = oldBlock.Index + 1
	newBlock.Timestamp = t.String()
	newBlock.BPM = BPM
	newBlock.PrevHash = oldBlock.Hash
	newBlock.Hash = calculateHash(newBlock)

	return newBlock, nil
}
```
Обратите внимание, что текущее время автоматически записывается в блок через ```time.Now()```. Так же обратите внимание, что была вызвана функция ```calculateHash```. В поле ```PrevHash``` скопировано значение хэша из предыдущего блока. ```Index``` просто увеличивается на единицу от значения из предыдущего блока.

Проверка блока
---
Теперь нам нужно написать функционал для проверки валидности предыдущих блоков. Мы делаем это проверяя ```Index```, что бы убедиться, что они увеличиваются так, как это ожидается. Мы так же проверяем, что бы ```PrevHash``` действиетльно совпадал с ```Hash``` предыдущего блока. И наконец, мы повторно вычисляем хэш текущего блока, что бы убедиться в его корректности. Давайте напишем функцию ```isBlockValid```, которая выполняет все эти действия и возвращает bool значение. Функция вернет ```true```, если все проверки пройдут верно:
```
func isBlockValid(newBlock, oldBlock Block) bool {
	if oldBlock.Index+1 != newBlock.Index {
		return false
	}

	if oldBlock.Hash != newBlock.PrevHash {
		return false
	}

	if calculateHash(newBlock) != newBlock.Hash {
		return false
	}

	return true
}
```
Что, если мы столкнемся с проблемой, когда два узла нашей blockchain экосистемы добавили блоки в свои цепочки, и мы получили их оба. Какой из них мы выберем, как правильный источник? Мы выбираем наиболее длинную цепь. Это классическая проблема в blockchain.

Итак, давайте убедимся, что новая цепочка, которую мы принимаем, длиннее текущей цепи. Если это так, мы можем перезаписать нашу цепочку новой, у которой есть новый блок или блоки.
<img src="https://habrastorage.org/webt/ur/5j/nn/ur5jnnqkq7vm3kcu66odvlk7fmo.png" alt="image"/>

Мы просто сравним длину срезов цепей:
```
func replaceChain(newBlocks []Block) {
	if len(newBlocks) > len(Blockchain) {
		Blockchain = newBlocks
	}
}
```
Если у Вас получилось, то можете похлопать себя по спине! Мы описали каркас функционала для нашего blockchain.

Теперь нам нужен удобный способ просмотра нашего blockchain и запись в него, в идеале в браузере, что бы мы могли похвастаться друзьям!

Web Server
---
Мы предполагаем, что вы уже знакомы с тем, как работают веб-серверы, и у вас есть немного опыта работы на Go.

Используем пакет ```Gorrila/mux```, который загрузили ранее. Создадим функцию ```run``` для запуска сервера и вызовем ее позже.
```
func run() error {
	mux := makeMuxRouter()
	httpAddr := os.Getenv("ADDR")
	log.Println("Listening on ", os.Getenv("ADDR"))
	s := &http.Server{
		Addr:           ":" + httpAddr,
		Handler:        mux,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}

	if err := s.ListenAndServe(); err != nil {
		return err
	}

	return nil
}
```
Обратите внимание, что порт конфигурируется из вашего ```.env```-файла, который мы создали ранее. Вызовем метод ```log.Println``` для вывода в консоль информации о запуске сервера. Мы настраиваем сервер и вызываем ```ListenAndServe```. Обычная практика в Go.

Теперь нам нужно написать функцию ```makeMuxRouter```, которая будет определять наши обработчики. Для просмотра и записи нашего blockchain в браузере нам хватит двух простых роутов. Если мы отправляем ```GET``` запрос на ```localhost```, то мы просматриваем нашу цепочку. Если отправляем ```POST``` запрос, то мы можем записывать данные.
```
func makeMuxRouter() http.Handler {
	muxRouter := mux.NewRouter()
	muxRouter.HandleFunc("/", handleGetBlockchain).Methods("GET")
	muxRouter.HandleFunc("/", handleWriteBlock).Methods("POST")
	return muxRouter
}
```

Обработчик ```GET``` запроса:
```
func handleGetBlockchain(w http.ResponseWriter, r *http.Request) {
	bytes, err := json.MarshalIndent(Blockchain, "", "  ")
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	io.WriteString(w, string(bytes))
}
```
Мы будем описывать blockchain в формате JSON, который можно будет просматривать в любом браузере по адресу ```localhost:8080```. Вы можете задать порт в файле ```.env```.

```POST``` запрос немножко сложнее и нам понадобится новая структура сообщений ```Message```.
```
type Message struct {
	BPM int
}
```
Код для обработчика записи в blockchain.
```
func handleWriteBlock(w http.ResponseWriter, r *http.Request) {
	var m Message

	decoder := json.NewDecoder(r.Body)
	if err := decoder.Decode(&m); err != nil {
		respondWithJSON(w, r, http.StatusBadRequest, r.Body)
		return
	}
	defer r.Body.Close()

	newBlock, err := generateBlock(Blockchain[len(Blockchain)-1], m.BPM)
	if err != nil {
		respondWithJSON(w, r, http.StatusInternalServerError, m)
		return
	}
	if isBlockValid(newBlock, Blockchain[len(Blockchain)-1]) {
		newBlockchain := append(Blockchain, newBlock)
		replaceChain(newBlockchain)
		spew.Dump(Blockchain)
	}

	respondWithJSON(w, r, http.StatusCreated, newBlock)

}
```
Причина, по которой мы использовали отдельную структуру сообщения, заключается в том, что тело ```POST``` запроса приходит в формате ```JSON``` и мы будем использовать его для записи новых блоков. Это позволяет нам отправить ```POST``` запрос следующего вида и наш обработчик заполнит оставшуюся часть блока за нас:
```
{"BPM":50}
```
```50``` - пример частоты пульса. Можете использовать своё значение пульса.

После декодирования тела запроса в структуру ```var m Message```, мы создадим новый блок, передавая предыдущий бок и новое значение пульса в функцию ```generateBlock```, которую мы писали ранее. Проведем быструю проверку, что бы убедиться в правильности нового блока функцией ```isBlockValid```.

*Примечания:*
* *```spew.Dump``` - удобная функция, которая красиво выводит структуры в консоль. Очень помогает в отладке.*
* *для тестирования запросов, нам нравится использовать Postman. ```curl``` так же хорошо справляется, если вы не можете уйти от терминала.*

Хочется получать уведомление, когда наши ```POST``` запросы успешны или завершились с ошибкой. Мы используем небольшую обертку, для получения результата. Помните, что в Go никогда не игнорируются ошибки.

```
func respondWithJSON(w http.ResponseWriter, r *http.Request, code int, payload interface{}) {
	response, err := json.MarshalIndent(payload, "", "  ")
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		w.Write([]byte("HTTP 500: Internal Server Error"))
		return
	}
	w.WriteHeader(code)
	w.Write(response)
}
```

Почти готово!
---

Давайте соединим все наработки в одной функции ```main```:
```
func main() {
	err := godotenv.Load()
	if err != nil {
		log.Fatal(err)
	}

	go func() {
		t := time.Now()
		genesisBlock := Block{0, t.String(), 0, "", ""}
		spew.Dump(genesisBlock)
		Blockchain = append(Blockchain, genesisBlock)
	}()
	log.Fatal(run())

}
```
Что здесь происходит?
* ```godotenv.Load()``` позволяет нам читать переменные из файла ```.env```
* genesisBlock - самая важная часть основной функции ```main```. Нам нужно проинициализировать первый блок, т.к. предыдущего блока еще не существует.

Все готово!
---
Весь код вы можете забрать с [github](https://github.com/mycoralhealth/blockchain-tutorial/blob/master/main.go)
Давайте проверим наш код.
Запускаем в терминале наше приложение ```go run main.go```
В терминале мы видим, что веб-сервер работает и мы получаем вывод нашего проинициализированного первого блока.
<img src="https://habrastorage.org/webt/be/jt/qr/bejtqr4zwjp7ei_u2qcnehkxra4.png" alt="image"/>

Теперь посетите [localhost:8080](localhost:8080). Как и ожидалось, мы видим первый блок.
<img src="https://habrastorage.org/webt/9s/vd/zu/9svdzuyfuhj1pk-kp58p1hr_gyc.png" alt="image"/>

Теперь давайте отправим ```POST``` запросы для добавления блоков. Используя Postman, мы собираемся добавить несколько новых блоков с различными значениями ```BPM```.

*curl команда (от переводчика):*
```
curl -X POST http://localhost:8080/ -H 'content-type: application/json' -d '{"BPM":50}'
```
<img src="https://habrastorage.org/webt/wj/df/ap/wjdfapcyu3jkbznctcerljwyjrs.png" alt="image"/>

Обновим нашу страничку в браузере. Теперь можно увидеть новые блоки в нашей цепочке. Новые блоки содержат ```PrevHash```  соответствуют ```Hash``` у старых блоков, как мы и ожидали!
<img src="https://habrastorage.org/webt/ek/az/yl/ekazylw-knpucadiu9rnle9nluq.png" alt="image"/>

В дальнейшем
---
Поздравляем! Вы только что создали свой blockchain с правильным хэшированием и блочной проверкой. Теперь Вы можете изучать более сложные проблемы blockchain, такие, как Proof of Work, Proof of Stake, Smart Contracts, Dapps, Side Chains и другие.

Данный урок не затрагивает такие темы, как новые блоки добавляются с помощью Proof of Work. Это будет отдельный урок, но существует множество blockchain и без механизмов Proof of Work. Сейчас все моделируется путем записи и просмотра данных blockchain на веб-сервере. В этом уроке нет составляющей P2P.

Если Вы хотите, что бы мы добавили механизм Proof of Work и работу по сети, вы можете сообщить об этом в [чате Telegram](https://t.me/joinchat/FX6A7UThIZ1WOUNirDS_Ew) или подписаться на нас в [Twitter](https://twitter.com/myCoralHealth)! Это лучшие способы связаться с нами. Мы ждем новых отзывов и новых предложений по урокам. Мы рады услышать Вас!

Чтобы узнать больше о Coral Health и о том, как мы используем blockchain в исследовательской работе по медицине, можете посетить наш [сайт](https://mycoralhealth.com/).

*P.S. Автор перевода будет благодарен за указанные ошибки и неточности перевода. [Оригинал](https://medium.com/@mycoralhealth/code-your-own-blockchain-in-less-than-200-lines-of-go-e296282bcffc) статьи.*
