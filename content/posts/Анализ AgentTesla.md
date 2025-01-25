+++
title = 'Анализ AgentTesla'
date = 2024-08-04T21:14:26+10:00
draft = false
+++

## Введение

AgentTesla — это тип вредоносного ПО, известный как Remote Access Trojan (RAT) и кейлоггер. Оно активно используется злоумышленниками для кражи конфиденциальной информации с заражённых систем.

## Общие сведения

**1. Тип:** Remote Access Trojan (RAT) и кейлоггер

**2. Основные характеристики:**
* **Шпионские возможности:** AgentTesla способен перехватывать и записывать нажатия клавиш, делать скриншоты, захватывать буфер обмена и собирать данные из различных приложений.
* **Кража данных:** Вредоносное ПО часто нацелено на кражу данных аутентификации из веб-браузеров, почтовых клиентов и других приложений.
* **Удалённое управление:** AgentTesla позволяет злоумышленникам удалённо управлять заражённой системой, выполняя команды и действия на целевом устройстве.

**3. Распространение:**
* **Фишинговые атаки:** AgentTesla часто распространяется через фишинговые электронные письма, содержащие вредоносные вложения или ссылки.
* **Эксплоиты:** Использование уязвимостей в программном обеспечении для загрузки и выполнения вредоносного кода.
* **Социальная инженерия:** Злоумышленники могут использовать методы социальной инженерии для обмана пользователей с целью установки вредоносного ПО.

**4. Вектор атаки:**
* **Вложения в электронных письмах:** Вредоносные документы, такие как файлы Word или Excel с вредоносными макросами, или архивы, содержащие вредоносные исполняемые файлы.
* **Ссылки на вредоносные сайты:** Ссылки в письмах, ведущие на сайты, которые автоматически загружают и запускают AgentTesla.

**5. Функции и возможности:**
* **Кейлоггер:** Записывает все нажатия клавиш, что позволяет злоумышленникам захватывать пароли и другую конфиденциальную информацию.
* **Скриншоты:** Делает скриншоты экрана заражённого компьютера.
* **Сетевые данные:** Собирает данные о сетевой активности, включая информацию о подключениях и передаваемых данных.
* **Данные из приложений:** Извлечение данных из  веб-браузеров (Chrome, Firefox, Internet Explorer), почтовых клиентов (Outlook, Thunderbird) и других приложений.
* **Экфильтрация данных:** Передача собранных данных на удалённый сервер злоумышленников через различные протоколы, включая SMTP, FTP и HTTP.

## Статический анализ

Даннные, полученые в DIE:

**1. Структура файла**
	**Размер файла:** 1.01 МБ  
	**Тип файла:** PE (Portable Executable)  
	**Компилятор**: AutoIt(3.XX)
	**Хэш SHA-256:** D4C4EE49A5CE076550C8305FCD63FE86707A251A2CA7D47C67D0DBEF66B2A1E3

**2. Секции файла**

| Offset   | Size     | Entropy | Status     | Name                 |
| -------- | -------- | ------- | ---------- | -------------------- |
| 00000000 | 00000400 | 2.93857 | not packed | PE Header            |
| 00000400 | 0008e000 | 6.67525 | packed     | Section(0)['.text']  |
| 0008e400 | 0002fe00 | 5.76325 | not packed | Section(1)['.rdata'] |
| 000be200 | 00005200 | 1.19645 | not packed | Section(2)['.data']  |
| 000c3400 | 00038400 | 7.78357 | packed     | Section(3)['.rsrc']  |
| 000fb800 | 00007200 | 6.78400 | packed     | Section(4)['.reloc'] |


 **2. Структура и импортируемые библиотеки**
 
Библиотеки:
- placeholder

**3. Основные функции и их анализ**

Основные функции:
- placeholder

**4. Анализ строк и конфигурационных данных**

Строки:

- placeholder

Конфигурационный блок:

* placeholder

**5. Обфускация**

Методы обфускации:
- Исполняемый файл упакован с помощью AutoIt и обфусцирован. С помощью AutoItRipper была извлечена обфусцированная конфигурация.



## Динамический анализ

**1. Сетевая активность**  
* placeholder

**2. Файловые операции**  
* placeholder

**3. Отладка**

- Отладка в x64dbg:
	1. Проверка **IsDebuggerPresent**, после чего выводится сообщение: *"This is a third-party compiled AutoIt Script."*
	2. Process hollowing:
		Создание легетимного процесса  
			1. AgentTesla создает  процесс `RegSvcs.exe` с флагом **CREATE_SUSPENDED**, используя следующую команду: placeholder
			2. Запись полезной нагрузки: placeholder
			3. Возобновление процесса с помощью **ResumeThread**: placeholder

- Извлечение конфигурации
		С помощью HollowsHunter из процесса **RegSvcs.exe** извлечена сборка .NET. В dnSpy отображаются классы и функции сборки в обфусцированном виде.
		
- Отладка извлеченной сборки в dnSpy
	1. Настройка HTTPS соединения
		- Установка поддерживаемых протоколов (SSL 3.0, TLS 1.0, TLS 1.1, TLS 1.2)
			```csharp
			if (num == 1)
			{
				ServicePointManager.SecurityProtocol = (SecurityProtocolType.Ssl3 | SecurityProtocolType.Tls | SecurityProtocolType.Tls11 | SecurityProtocolType.Tls12);
				num = 2;
			}
			```
		- Устанавливается делегат, который всегда возвращает true, подтверждая любую проверку сертификатов.
	
			```csharp
			 if (num == 2)
			{
				Delegate serverCertificateValidationCallback = ServicePointManager.ServerCertificateValidationCallback;
				if (TIih8WcH.CS$<>9__CachedAnonymousMethodDelegate1 == null)
				{
					TIih8WcH.CS$<>9__CachedAnonymousMethodDelegate1 = new RemoteCertificateValidationCallback(TIih8WcH.jgbfREqD);
				}
				ServicePointManager.ServerCertificateValidationCallback = (RemoteCertificateValidationCallback)Delegate.Combine(serverCertificateValidationCallback, TIih8WcH.CS$<>9__CachedAnonymousMethodDelegate1);
				num = 3;
			}
			```
		- Функция проверки сертификатов 
		
			```csharp
			private static bool jgbfREqD(object mUnuRbJ3, X509Certificate yGOKueQ, X509Chain frxz, SslPolicyErrors WSz)
			{
				int num = 0;
				do
				{
					if (num == 0)
					{
						num = 1;
					}
				}
				while (num != 1);
				return true;
			}
			```
	1. Завершение всех процессов, кроме текущего 
		```csharp
		string processName = Process.GetCurrentProcess().ProcessName;
				int id = Process.GetCurrentProcess().Id;
				Process processesByName = Process.GetProcessesByName(processName);
				foreach (Process process in processesByName)
				{
					if (process.Id != id)
					{
						process.Kill();
					}
				}
		```
	1. Генерация идентификатора машины
		- Функция вызывает другие функции для получения серийного номера материнской платы, идентификатора процессора и MAC-адреса сетевого адаптера. Затем она объединяет их и вычисляет MD5-хэш от этой строки. В случае ошибки возвращается строка "None".
			
			```csharp
			text = fHfXQzVTqo.ouKT(MD5.Create(), fHfXQzVTqo.wENtroM1A() + fHfXQzVTqo.gwVUs7kYe() + fHfXQzVTqo.kz1WOlEk7TS());
			```
	1. Настройка автозагрузки 
	   - Получение пути к загруженному файлу сборки 
		```csharp
	   Stu4Un2.AsmFilePath = Assembly.GetExecutingAssembly().Location;
	   ```
	   - Получение пути до директории хранения данных приложения (окружение + имя поддиректории)
	   ```csharp
	    Stu4Un2.StartupDirectoryPath = Path.Combine(Environment.GetEnvironmentVariable(Stu4Un2.StartupEnvName), Stu4Un2.StartupDirectoryName);
		```  
		- Получение полного пути к файлу автозапуска 
		```csharp
		Stu4Un2.AppStartupFullPath = Path.Combine(Stu4Un2.StartupDirectoryPath, Stu4Un2.StartupInstallationName);
		```
		
	1. Взаимодействие с C2
		-  Формирование строки "ИмяПользователя/ИмяКомпьютера"
		```csharp
			Stu4Un2.ThisComputerName = SystemInformation.UserName + "/" + SystemInformation.ComputerName;
		```
		- Создание и настройка веб-запроса
			- Выполняется GET-запрос к "https://api.ipify.org", и возвращает содержимое ответа в виде строки. Если запрос успешен и сервер вернул статус "OK", то возвращается тело ответа. В противном случае возвращается пустая строка.
		```csharp
		HttpWebRequest httpWebRequest = (HttpWebRequest)WebRequest.Create(Stu4Un2.IpApi);
		httpWebRequest.Credentials = CredentialCache.DefaultCredentials;
		httpWebRequest.KeepAlive = true;
		httpWebRequest.Timeout = 10000;
		httpWebRequest.AllowAutoRedirect = true;
		httpWebRequest.MaximumAutomaticRedirections = 50;
		httpWebRequest.Method = "GET";
		httpWebRequest.UserAgent = Stu4Un2.PublicUserAgent;
		```
	1. Проверки на песочницу
		- Проверка наличия отладчика
		```csharp
		n1uhLjf.CheckRemoteDebuggerPresent(Process.GetCurrentProcess().Handle, ref flag);
		```
		- Проверка, находится ли IP-адрес компьютера в диапазоне хостинг-провайдеров
		```csharp
		string text = new WebClient().DownloadString("http://ip-api.com/line/?fields=hosting");
					return text.Contains("true");
		```
	1. Проверки времени сна
		- Некоторые виртуализированные среды и песочницы пытаются замедлить выполнение программы, чтобы затруднить анализ её поведения. Например, они могут искусственно увеличивать время, необходимое для выполнения команд, чтобы имитировать реальную производительность. Если система замедлит выполнение команды Sleep(10), но не учтет реального времени, то произойдет расхождение во времени, и метод выявит это несоответствие.
		```csharp
		long ticks = DateTime.Now.Ticks;
		Thread.Sleep(10);
		if (DateTime.Now.Ticks - ticks < 10L)
		{
		    return true;
		}
		```
	1. Проверка модулей в памяти, связанных с виртуализацией
		- SbieDll.dll (VmWare)
	    - Snxhk.dll, Cmdvrt32.dll (Hyper-V)
	    - Cmdvrt32.dll (Windows-Server)
	    - Sf2.dll (IO)
    1. Проверки производителя компьютера и видеокарты
		- Поиск объектов ManagementObjectSearcher:
		- В цикле выполняется поиск объектов ManagementObjectSearcher, которые соответствуют компьютерам, работающим в виртуализированной среде или песочнице.
		- Проверка производителей и моделей:
		    - Для каждого найденного объекта проверяется, содержит ли его производитель строку "Microsoft Corporation" и модель "VIRTUAL". Если это так, метод возвращает true, указывающий на работу в виртуализированной среде.
		    - Также проверяется наличие производителей "VMware" и моделей "VirtualBox".
		- Проверка видеокарт:
		- Если объекты имеют названия, содержащие "VMware" или "VBox", метод возвращает true, предполагающий работу в виртуализированной среде.
```csharp
	    managementObjectSearcher = new ManagementObjectSearcher("Select * from Win32_ComputerSystem");
	  	using (ManagementObjectCollection managementObjectCollection = managementObjectSearcher.Get())
				{
					foreach (ManagementBaseObject managementBaseObject in managementObjectCollection)
					{
						if ((managementBaseObject["Manufacturer"].ToString().ToLower() == "microsoft corporation" && managementBaseObject["Model"].ToString().ToUpperInvariant().Contains("VIRTUAL")) || managementBaseObject["Manufacturer"].ToString().ToLower().Contains("vmware") || managementBaseObject["Model"].ToString() == "VirtualBox")
						{
							return true;
						}
					}
				}
			}
			catch
			{
				return true;
			}
			finally
			{
				if (managementObjectSearcher != null)
				{
					((IDisposable)managementObjectSearcher).Dispose();
				}
			}
			foreach (ManagementBaseObject managementBaseObject2 in new ManagementObjectSearcher("root\\CIMV2", "SELECT * FROM Win32_VideoController").Get())
			{
				if (managementBaseObject2.GetPropertyValue("Name").ToString().Contains("VMware") && managementBaseObject2.GetPropertyValue("Name").ToString().Contains("VBox"))
				{
					return true;
				}
			}
			return false;
```
Далее, перейдя в класс **qXK**, обфускация сильно увеличилась: 

![Animation.gif](/Animation.gif)

Попытка воспользоваться **de4dot** с его дефолтными параметрами не оказала никакого влияния на код.
После этого, я наткнулся на  [статью](https://ryan-weil.github.io/posts/AGENT-TESLA-2), в которой описывается написание реализации деобфускатора для  для de4dot. 
- Краткое описание метода  
	- В статье показан процесс "выпрямления" метода, т.е. избавления от условий и переходов.    
- Анализ потока управления  
	В целом, принцип обфускации этой (и не только, в данной сборке) функции заключается в изменении потока управления и ветвления с помощью if-ов. Общий алгоритм:  
	1. задание начального значения num = 0  
	2. переход к следующей инструкции (условию if)
	3. проверка, соответвует ли num числу i  
	- если да, то:
		- вход в тело условия
		- выполнение инструкции  
		- num = новое число  
	4. переход в п.2   
![Подпись](/aget_tesla.drawio.png)  
Результат

![flat.png](/flat.png)
## Цепочка заражения
Placeholder
## Indicators of Compromise
| Категория |                              SHA256                              |       Описание       |
| :-------: | :--------------------------------------------------------------: | :------------------: |
|   Файлы   | d4c4ee49a5ce076550c8305fcd63fe86707a251a2ca7d47c67d0dbef66b2a1e3 | Изначальный ZIP-файл |
## Mitre Attack TTPs
Placeholder
## Ссылки
* https://app.any.run/tasks/ee2f1b8a-8be0-4444-9d1a-fbcb87131191
* https://bazaar.abuse.ch/sample/d4c4ee49a5ce076550c8305fcd63fe86707a251a2ca7d47c67d0dbef66b2a1e3/