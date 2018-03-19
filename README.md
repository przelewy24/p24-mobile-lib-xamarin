# Dokumentacja biblioteki Przelewy24 - Xamarin

Ogólne informacje o działaniu bibliotek mobilnych w systemie Przelewy24 znajdziesz pod adresem:

- [https://github.com/przelewy24/p24-mobile-lib-doc](https://github.com/przelewy24/p24-mobile-lib-doc)

Przykład implementacji biblioteki:

- [https://github.com/przelewy24/p24-mobile-lib-xamarin-example](https://github.com/przelewy24/p24-mobile-lib-xamarin-example)

## 1. Konfiguracja projektu

### Dodawanie biblioteki

W Visual Studio należy dodać referencję (References -> Add References) do biblioteki P24Lib (plik P24Lib.dll)

### Dodanie widoku TransferPage
W celu poprawnego działania biblioteki należy utworzyć widok TransferPage, który w code behind zawierał będzie:
```java
public partial class TransferPage : ContentPage
{
	public TransferPage(TrnDirectParams transactionParams)
	{
		var content = TransferPageHelper.ContentForTrnDirect(transactionParams);
		content.AddNavigating(WebView_Navigating);
		Content = content;
	}

	public TransferPage(TrnRequestParams transactionParams)
	{
		var content = TransferPageHelper.ContentForTrnRequest(transactionParams);
		content.AddNavigating(WebView_Navigating);
		Content = content;
	}

	public TransferPage(ExpressParams transactionParams)
	{
		var content = TransferPageHelper.ContentForExpress(transactionParams);
		content.AddNavigating(WebView_Navigating);
		Content = content;
	}

	private void WebView_Navigating(object sender, WebNavigatingEventArgs e)
	{
		if (TransferPageHelper.IfTransactionFinished(e))
		{
			Navigation.PopAsync();
		}
	}

	protected override bool OnBackButtonPressed()
	{
		if (TransferPageHelper.CanMoveToBankList())
		{
			var newContent = TransferPageHelper.GetContentForBack();
			newContent.Navigating += WebView_Navigating;
			Content = newContent;
			return true;
		}
		else
		{
			return base.OnBackButtonPressed();
		}
	}
}
```

## 2. Wywołanie transakcji trnDirect

W tym celu należy stworzyć obiekt klasy TrnDirectParams i przy okazji ustawić parametry transakcji, podając Merchant Id i klucz do CRC:

```java
var payment = new TrnDirectParams()
            {
                SessionId = XXXXXXXXXXXXX,
                MerchantId = XXXXX,
                Crc = XXXXXXXXXXXXX,
                Amount = 1,
                Language = "pl",
                Currency = "PLN",
                Address = "Ulica testowa",
                City = "Poznan",
                Zip = "61-600",
                Client = "Jan Kowalski",
                Country = "PL",
                Email = "info@przelewy24.pl",
                Phone = "23566345"
            };
```

Parametry opcjonalne:

```java
payment.UrlStatus = "https://XXXXXX";
payment.Method = 25;
payment.TimeLimit = 90;
payment.Channel = 1;
payment.TransferLabel = "transfer label";
payment.Shipping = 0;
```

Opcjonalne można ustawić wywołanie transakcji na serwer Sandbox:

```java
payment.SetSandbox(true)
```

Mając gotowe obiekty konfiguracyjne możemy przystąpić do wywołania widoku dla transakcji. Uruchomienie wygląda następująco:

```java
var payment = GetTransferParams();
var transferPage = new TransferPage(payment.SetSandbox(setSandbox));
transferPage.Disappearing += (sender2, e2) => { TransferPageResult(); };
await Navigation.PushAsync(transferPage);
```

Aby obsłużyć rezultat transakcji należy rozszerzyć metodę `TransferPageResult`:

```java
private void TransferPageResult()
{
	var result = Result.Get();

	if (result != null)
	{
		if (result.IsSuccess)
		{
			// success
		}
		else
		{
			// error
			var errorCode = result.ErrorCode;
		}
	}
}
```
`TransferPage` zwraca tylko informację o tym, że transakcja się zakończyła. Nie zawsze oznacza to czy transakcja jest zweryfikowana przez serwer partnera, dlatego za każdym razem po uzyskaniu statusu `isSuccess` aplikacja powinna odpytać własny backend o status transakcji.

## 3. Wywołanie transakcji trnRequest

Podczas rejestracji transakcji metodą "trnRegister" należy podać dodatkowe parametry:
- `p24_mobile_lib=1`
- `p24_sdk_version=X` – gdzie X jest wersją biblioteki mobilnej otrzymana w wywołaniu metody `P24SdkVersion.value()`

Dzięki tym parametrom system Przelewy24 będzie wiedział że powinien traktować transakcję jako mobilną. Token zarejestrowany bez tego parametru nie zadziała w bibliotece mobilnej (wystąpi błąd po powrocie z banku i okno biblioteki nie wykryje zakończenia płatności).

**UWAGA!**

 > Rejestrując transakcję, która będzie wykonana w bibliotece mobilnej należy
pamiętać o dodatkowych parametrach:
- `p24_channel` – jeżeli nie będzie ustawiony, to domyślnie w bibliotece pojawią się
formy płatności „przelew tradycyjny” i „użyj przedpłatę”, które są niepotrzebne przy płatności mobilnej. Aby wyłączyć te opcje należy ustawić w tym parametrze flagi nie
uwzględniające tych form (np. wartość 3 – przelewy i karty, domyślnie ustawione w
bibliotece przy wejściu bezpośrednio z parametrami)
- `p24_method` – jeżeli w bibliotece dla danej transakcji ma być ustawiona domyślnie
dana metoda płatności, należy ustawić ją w tym parametrze przy rejestracji
- `p24_url_status` - adres, który zostanie wykorzystany do weryfikacji transakcji przez serwer partnera po zakończeniu procesu płatności w bibliotece mobilnej

Należy ustawić parametry transakcji podając token zarejestrowanej wcześniej transakcji, opcjonalnie można ustawić serwer sandbox oraz konfigurację banków:

```java
var token = "XXXX-XXXX-XXXX-XXXX"; // transaction token
var transferPage = new TransferPage(new TrnRequestParams(token).SetSandbox(setSandbox));
transferPage.Disappearing += (sender2, e2) => { TransferPageResult(); };
await Navigation.PushAsync(transferPage);
```

Rezultat transakcji należy obsłużyć identycznie jak dla wywołania "trnDirect".

## 4. Wywołanie transakcji Ekspres

Należy ustawić parametry transakcji podając url uzyskany podczas rejestracji transakcji w systemie Ekspres. Transakcja musi być zarejestrowana jako mobilna.

```java
var url = "https://XXX"; // url to express transaction
var transferPage = new TransferPage(new ExpressParams(url));
transferPage.Disappearing += (sender2, e2) => { TransferPageResult(); };
await Navigation.PushAsync(transferPage);
```

Rezultat transakcji należy obsłużyć identycznie jak dla wywołania "trnDirect".

## 5. Wywołanie transakcji z Pasażem 2.0

Należy ustawić parametry transakcji identycznie jak dla wywołania "trnDirect", dodając odpowiednio przygotowany obiekt koszyka:

```java
payment.PassageCart = new PassageCart();

var item = new PassageItem()
{
	Name = "Product 1",
	Description = "description 1",
	Quantity = 1,
	Price = 100,
	Number = 1,
	TargetAmount = 100,
	TargetPosId = 51986
};

payment.PassageCart.AddItem(item);

item = new PassageItem()
{
	Name = "Product 2",
	Description = "description 2",
	Quantity = 1,
	Price = 100,
	TargetAmount = 100,
	TargetPosId = 51987
};

payment.PassageCart.AddItem(item);
```

Wywołanie transakcji oraz parsowanie wyniku jest realizowane identycznie jak dla wywołania "trnDirect".
