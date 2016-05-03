# Передаем объекты между процессами в MS-MPI

Для тех, кто работал с библиотеками, реализующими стандарт MPI, не секрет, что существует ряд ограничений, накладываемых на типы данных,
которые можно пересылать между процессами. Например, в MPI вам не разрешит без проблем передать вектор значений, строку или любой другой
объект. Однако, данное ограничение можно преодолеть воспользовавшись инструментами сериализации объектов в массив байтов. Именно о такой 
технике и пойдет речь ниже.

Изобретать велосипед сериализации я не стану, да и задача эта весьма нетривиальная. Поэтому в статье для этих целей будет использована 
библиотека [Boost.Serialization](http://www.boost.org/doc/libs/1_60_0/libs/serialization/doc/index.html). Она обладает всеми необходимыми инструментами, а также сам Boost реализует классы расширения для стандартных
STL контейнеров, таких как vector, set и другие, позволяя при этом без проблем использовать для их передачи изложенный в
статье метод. Скачать буст можно с [официального сайта](http://www.boost.org/), либо же установить при помощи [Nuget-пакета](https://www.nuget.org/packages/boost) 
в Visual Studio.

Перейдем к самой проблеме, пусть у нас есть два процесса, и нам необходимо реализовать взаимодействие между ними по типу точка-точка.
То есть на отправляющей стороне будем вызывать операцию `MPI_Send`, а на принимающей `MPI_Recv`. Звучит просто, не так ли? Тогда перейдем
к самой реализации.

В качестве рассматриваемого
класса для передачи между процессами возьмем следующий:

``` c++
class Person {
  public:
    Person(int age, string name)
      : _age(age), _name(name)
    { }
    Person()
      : _age(-1), _name("unknown")
    { }
  public:
    int getAge() const { return _age; }
    string getName() const { return _name; }
  private:
    int _age;
    string _name;
};
```

Следующим делом необходимо реализовать функции, которые будут выполнять сериализацию и десериализацию объекта. В принципе можно было
бы сделать некоторый отдельный класс, который бы этим занимался, но дабы не усложнять пример сделаем эти функции внешними. 
Для начала разберемся с тем, какие нужны заголовки из буста. А нужны нам следующие:

``` c++
#include <boost/archive/binary_iarchive.hpp>
#include <boost/archive/binary_oarchive.hpp>
#include <boost/serialization/set.hpp>
#include <boost/iostreams/stream_buffer.hpp>
#include <boost/iostreams/stream.hpp>
#include <boost/iostreams/device/back_inserter.hpp>
```

Теперь запишем наши функции и заодно станет видно, где и как применяется каждый из включенных хедеров. Cначала запишем функцию 
сериализации шаблонного объекта типа `<T>`

``` c++
template<typename T>
std::string serialize(const T& what) {
	std::string serialized_object;
	boost::iostreams::back_insert_device<std::string> inserter(serialized_object);
	boost::iostreams::stream<boost::iostreams::back_insert_device<std::string>> stream(inserter);
	boost::archive::binary_oarchive archive(stream);
	archive << what;
	stream.flush();
	return serialized_object;
}
``` 
Давайте разберем, что же происходит в самой функции. Как видно из сигнатуры, объект мы будем сериализовать в строку. В принципе
можно было бы использовать и динамический массив char-ов, но для этого пришлось бы через параметр функци возвращать длину 
сериализованного сообщения. Также в коде видно связывание различных слоев представления данных. Бинарный архив передает сериализованные
данные в поток строки, которая в свою очередь связана с исходной строкой через `back_insert_device`. Строка `archive << what` вызывает
метод `serialize` у нашего объекта, который определяет какие из полей должны быть сохранены, для восстановления состояния. В данный момент
в классе Person нет метода `serialize`. Мы вернемся к нему позже.

Теперь рассмотрим функцию десериализации объекта
```c++
T deserialize(char* data, int length) {
	boost::iostreams::basic_array_source<char> device(data, length);
	boost::iostreams::stream<boost::iostreams::basic_array_source<char> > s(device);
	boost::archive::binary_iarchive archive(s);
	T obj;
	archive >> obj;
	return obj;
}
```
Из кода видно, что применяется механизм обратный сериализации. Теперь мы инициализируем `binary_iarchive` потоком, содержащим состояние
нашего объекта. В свою очередь уже архив берет на себя обязанности конвертации потока char-ов в исходный объект.

Добавим последний штрих для успешной сериализации нашего объекта, а именно метод `serialize`. Запишем его следующим образом:
```c++
class Person {
...
public:
	template<class Archive>
	void serialize(Archive &ar, const unsigned int version) {
		ar & _age;
		ar & _name;
	}
...
};
```

Суть метода серилизации сводится к тому чтобы записать все необходимые поля объекта в архив. В нашем случае это поля _age и _name.

Теперь с сериализацией точно всё. Осталось дописать несколько MPI-функций, которые будут выполнять передачу и прием объекта в 
пространстве определенного коммуникатора. 

Реализуем их для типа передачи точка-точка. Также, так как мы не можем заранее определить на принимающей стороне размер сериализованного
сообщения, разобьем операцию передачи объекта на две суб-операции прием-передача. В первой части будем выполнять передачу длины сообщения, а во второй
сам массив данных.

Запишем функцию для передачи объекта
```c++
template<typename T, ENABLE_IF_CLASS(T)>
void MPI_SendObject(const T& what, int destination, int tag, MPI_Comm comm)
{
	std::string serialized = serialize(what);
	int length = serialized.length();
	MPI_Send(&length, 1, MPI_INT, destination, 666, comm);
	char* data = const_cast<char*>(serialized.data());
	MPI_Send(static_cast<void*>(data), serialized.length(), MPI_BYTE, destination, tag, comm);
}
```
>__Важно!__ Тег сообщения, в котором передается длина сериализованного объекта должен отличаться от 
тега сообщения, в котором передается сам массив данных

Аналогично запишем функцию для получения объекта
```c++
template<typename T>
T MPI_RecvObject(int from, int tag, MPI_Comm comm) {
	int length;
	MPI_Recv(&length, 1, MPI_INT, from, 666, comm, MPI_STATUSES_IGNORE);
	char* str_des = new char[length];
	MPI_Recv(str_des, length, MPI_BYTE, from, tag, comm, MPI_STATUS_IGNORE);
	T obj = deserialize<T>(str_des, length);
	delete[] str_des;
	return obj;
}
```

Вот собственно и всё! Теперь можно передавать объекты между процессами. Однако, стоит заметить, что данная операция 
является достаточно дорогостоящей в плане процессорного времени. И стоит использовать данный подход только в случае, если это 
действительно необходимо. 

Короткий пример использования:
```c++
void main(int argc, char** argv) {
	MPI_Init(&argc, &argv);
	int size, rank;
	MPI_Comm_size(MPI_COMM_WORLD, &size);
	MPI_Comm_rank(MPI_COMM_WORLD, &rank);
	
	if (rank == 0) {
		Person person(19, "John");
		std::cout << "[" << rank << "] Sending object Person {" << person.getAge() 
				  << "; " << person.getName() << "}\n";
		MPI_SendObject(person, 1, 1, MPI_COMM_WORLD);
	}

	if (rank == 1) {
		Person person = MPI_RecvObject<Person>(0, 1, MPI_COMM_WORLD);
		std::cout << "[" << rank << "] Received object Person {" << person.getAge()
				  << "; " << person.getName() << "}\n";
	}
	MPI_Finalize();
}
```
	

