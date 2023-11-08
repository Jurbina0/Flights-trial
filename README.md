# Flights-trial
6.11.23 (posiblemente)
## Notas

Con este [url](https://www.bannerbear.com/blog/pyppeteer-how-to-use-puppeteer-in-python-installation-guide-and-examples/#getting-started-with-pyppeteer) se sigue un ejemplo.

Ha hecho falta instalar 

+ librería pyppeteer
+ librería asyncio
+ python de versión mayor a 3.6
+ al usar jupyter: nest_asyncio.

jupyer por defecto no permite la anidación de bucles de eventos
+ websockets versión 10.4: debe de ser menor que la 11 para que funcione pyppeter
+ chromium: instalación manual

7.11.23
### Observaciones
Al entrar en [skyscanner](https://www.skyscanner.es/) me ha pedido mantener pulsado un botón que estaba en un snippet con advertencia de que era para impedir bots.
Luego he debido de aceptar las cookies. 

## Con subprocess

+ librería subprocess de python
+ node.js
+ puppeteer
+ chromium (requiere chrome)

Para guardar desde javascript un archivo csv:
+ paquete fs o fse-extra (que incluye fs)

## Grabación
+ Prueba con obs
+ Guardado en csv de horas en [wikipedia](https://es.wikipedia.org/wiki/Hora)

8.11.23
## Guardar en csv 3 columnas de números

De la web de wikipedia antes descrita se han hecho 3 columnas, cada fila distinguida por saltos de línea '\n' y elementos de la columna con ','

```
const puppeteer = require('puppeteer');
const fse = require('fs-extra');

(async () => {
    const browser = await puppeteer.launch({ headless: false });  // Modo no headless para ver el navegador
    const page = await browser.newPage();
    await page.goto(process.argv[2], { waitUntil: 'networkidle0' });
    // de aquí habrás obtenido la página web a scrapear

    // Extraer números de ul, li
    const numbers = await page.$$eval('ul > li', listItems => {
        return listItems.map(li => {
            const text = li.innerText;
            const matches = text.match(/\b\d+(?:\s\d+)*\s+horas\b/g);
            return matches ? matches.map(match => {
                const numberString = match.replace(/\s+horas\b/, '').replace(/\s/g, '');
                return parseInt(numberString, 10);
            }) : []; // de aquí saldrán integers
        }).flat();
    });
    await browser.close();
    // hasta aquí lo que es extraer info de la web
    // para dividir en 3 columnas del csv, se van completando por filas
    const rows = [];
    for (let i = 0; i < numbers.length; i += 3) {
        // Crea un arreglo para la fila actual.
        const row = [
            numbers[i], // Primera columna
            numbers[i + 1] || '', // Segunda columna (o cadena vacía si no existe)
            numbers[i + 2] || '', // Tercera columna (o cadena vacía si no existe)
        ];
        // Agrega la fila al arreglo de filas.
        rows.push(row.join(',')); // push es como append del python pero devuelve la longitud
    }
    // si quisiéramos cabecera vendría aquí
    const header = 'Fecha '
    const csvContent = rows.join('\n') // separación entre filas por saltos de línea
    // Guardar en un archivo CSV
    const csvPath = process.argv[3];
    // outputfile y fs-extra crean directorio si no existe
    await fse.outputFile(csvPath, csvContent);
})();
```

### Resultado
![Captura desde 2023-11-09 00-21-50](https://github.com/Jurbina0/Flights-trial/assets/147608303/8db88e6c-b2c5-4d3b-aa4b-b12aa8192e34)


##  Javascript code
+ Con "inspeccionar" en google chrome, seleccionar "Elements" y pasando el cursor encima del elemento de interés podemos encontrar las líneas del html concretas de lo que seleccionamos.
+ Hemos visto:
    + CheckBox, Button, Input dentro de span
    + [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction) en págines/archivos HTML, XML. Nodos vs objetos.
        + Es un API; habitualmente se conecta con JS, se puede Python (p.e. [xml.dom.minidom](https://docs.python.org/3/library/xml.dom.minidom.html)
        + DOM tree: [Document](https://developer.mozilla.org/en-US/docs/Web/API/Document)
        + Window: [Window](https://developer.mozilla.org/en-US/docs/Web/API/Window) Ventana que contiene el DOM tree
    + Definir las acciones del scraper mediante selectores (nodos?) [XPath](https://www.scrapingbee.com/blog/practical-xpath-for-web-scraping/)
        + CSS selector
        + Xpath selector
    + Elegir los elementos usando ruta XPath (usando 'id'), absoluta XPath (poco robusta, alterando la web deja de funcionar), ruta JS

### Observaciones II
Al entrar inicialmente en [skyscanner](https://www.skyscanner.es/) directamente aceptando cookies se podía.

```{JS}
  const acceptCookiesButton = await page.waitForSelector('#acceptCookieButton', { visible: true });
    // Si el botón está disponible, hacer clic en él
    if (acceptCookiesButton) {
        await acceptCookiesButton.click();
    }
```
![skyscannercookie](https://github.com/Jurbina0/Flights-trial/assets/147608303/1d38384c-0d35-4c60-8496-e908dc516fb4)

Después de varios intentos reapareció lo de mantener pulsado.
Descartando cuando ya se debía de mantener pulsado:

+ hemos conseguido seleccionar casilla de vuelo directo
```

    // ahora aparecerá la ventana con la opción de elegir el vuelo
    // seleccionamos el checkBox de vuelo directo mediante XPath
    const checkBoxXPath = '/html/body/div[1]/div/div[1]/div/div/main/div[1]/div/div[2]/div/div[5]/label/input';
    const [checkBox] = await page.$x(checkBoxXPath);
    if (checkBox) {
        await checkBox.click();
    }
```
![skyscanner_checkbox](https://github.com/Jurbina0/Flights-trial/assets/147608303/e7784b99-8997-4376-8678-e73a8228cd13)

+ escribir destino 'Ámsterdam Schiphol (AMS)'
```
    // seleccionamos y escribimos "Ámsterdam Schiphol (AMS)" en destino
    const inputDestinationSelector = '//*[@id="destinationInput-label"]/div/div'; //ruta XPath
    await page.waitForXPath(inputDestinationSelector, { visible: true, timeout: 20000 }); 
    // Esperar a que el elemento sea visible
    const [inputDestination] = await page.$x(inputDestinationSelector);
    if (inputDestination) {
        await inputDestination.click(); // Hacer clic en el campo para asegurarse de que se puede escribir en él
        await page.keyboard.type('Ámsterdam Schiphol (AMS)'); // Escribir el texto
        // await page.keyboard.press('Enter'); // Presionar Enter
    }
```
![skyscanner_amsterdam](https://github.com/Jurbina0/Flights-trial/assets/147608303/c1cc5c4e-599c-4b63-a1c0-c2a7c3280536)

## **Problema**: Calendario dinámico
    + Necesario/simplificaría definir función navegar mes, año; función seleccionar día
    + no funcionaba a la hora de seleccionar casilla

```
const dateSelectorButtonSelector = '#app-root > div:nth-child(1) > div > div > main > div.Homepage_searchControlsContainer__ZWI2N > div > div.SearchControls_container__MDdjN > div > div.DateSearchControlsGroup_desktopDatesContainer__ZWZkZ.DateSearchControlsGroup_flexibleDateContainer__ZjhhN > div:nth-child(1) > div > button';
    await page.click(dateSelectorButtonSelector);
    // Esperar a que el calendario esté visible
    await page.waitForSelector('#app-root > div:nth-child(1) > div > div > main > div.Homepage_searchControlsContainer__ZWI2N > div > div.SearchControls_container__MDdjN > div > div.DateSearchControlsGroup_desktopDatesContainer__ZWZkZ.DateSearchControlsGroup_flexibleDateContainer__ZjhhN > div.SearchControlSelectorPopOver_container__ZTFjM.DateSearchControlsGroup_newDatesSelectorPopover__Y2YyN.DateSearchControlsGroup_datesSelectorPopover__NGVhM');

    const dateButtonSelector = '#app-root > div:nth-child(1) > div > div > main > div.Homepage_searchControlsContainer__ZWI2N > div > div.SearchControls_container__MDdjN > div > div.DateSearchControlsGroup_desktopDatesContainer__ZWZkZ.DateSearchControlsGroup_flexibleDateContainer__ZjhhN > div.SearchControlSelectorPopOver_container__ZTFjM.DateSearchControlsGroup_newDatesSelectorPopover__Y2YyN.DateSearchControlsGroup_datesSelectorPopover__NGVhM > div > div > div.DatePickerDesktop_form__NmY5M > div.CustomCalendar_customCalendar__ODhkY > div.CustomCalendar_calendarsContainer__NTQyN > div:nth-child(1) > div > div.CustomCalendar_weekContainer__NmVhM > div:nth-child(3) > div:nth-child(7) > div > button';
    await page.click(dateButtonSelector);
```


