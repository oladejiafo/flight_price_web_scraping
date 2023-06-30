# Flight Scraper: Automated Flight Data Extraction from Kayak

## Flight Scraper Documentation

## Introduction
This documentation provides an overview of the flight scraper script developed using Selenium and BeautifulSoup. The script extracts flight information, including prices, timings, and stopovers, from the Kayak website. The scraped data is stored in a Pandas DataFrame and saved as CSV files. Additionally, the script sends an email with the aggregated flight results as an attachment.

## Prerequisites
Before running the flight scraper script, ensure that the following dependencies are installed:
- Selenium
- BeautifulSoup
- Pandas
- NumPy
- smtplib (for sending emails)

## Setup
1. Clone the GitHub repository containing the flight scraper code.
2. Install the necessary dependencies using pip or another package manager.
3. Set up the Chrome WebDriver. Download the appropriate version of the ChromeDriver and update the path in the script to the WebDriver executable.
4. Configure the email settings. Update the sender email, receiver email, and email credentials in the script.

## Usage
1. Open the flight scraper script in a Python editor or IDE.
2. Modify the script variables to specify the desired origin, destinations, start dates, and other parameters.
3. Run the script to initiate the scraping process.
4. Monitor the script's progress as it retrieves flight data from Kayak.
5. Once the scraping is complete, the flight details will be saved in a CSV file named "flight_results.csv", and the aggregated results will be saved in "flight_results_agg.csv".
6. An email containing the aggregated flight results will be sent to the specified email address.

## Code Explanation
```python
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait 
from selenium.webdriver.support import expected_conditions
from selenium.webdriver.chrome.service import Service
from bs4 import BeautifulSoup
import re
import pandas as pd
import numpy as np
from datetime import date, timedelta, datetime
import time

import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.base import MIMEBase
from email import encoders

# The flight scraper function
def scrape(origin, destination, startdate, days, requests):
        global results
    
    enddate = datetime.strptime(startdate, '%Y-%m-%d').date() + timedelta(days)
    enddate = enddate.strftime('%Y-%m-%d')
    url = "https://www.kayak.com/flights/" + origin + "-" + destination + "/" + startdate + "?sort=bestflight_a&attempt=1&lastms=1688125874445"
  
    print("\n" + url)

    chrome_options = webdriver.ChromeOptions()
    agents = ["Firefox/66.0.3","Chrome/73.0.3683.68","Edge/16.16299"]
    print("User agent: " + agents[(requests%len(agents))])
    chrome_options.add_argument('--user-agent=' + agents[(requests%len(agents))] + '"')    
    chrome_options.add_experimental_option('useAutomationExtension', False)

    service = Service("/usr/local/bin/chromedriver.exe")
    
    driver = webdriver.Chrome(service=service, options=chrome_options, desired_capabilities=chrome_options.to_capabilities())
    
    driver.implicitly_wait(20)
    driver.get(url)

    #Check if Kayak thinks that we're a bot
    time.sleep(5) 
    soup=BeautifulSoup(driver.page_source, 'lxml')

    if soup.find_all('p')[0].getText() == "Please confirm that you are a real KAYAK user.":
        print("Kayak thinks I'm a bot, which I am ... so let's wait a bit and try again")
        driver.close()
        time.sleep(20)
        return "failure"

    time.sleep(20) #wait 20sec for the page to load
    
    soup=BeautifulSoup(driver.page_source, 'lxml')
    
    #get the arrival and departure times
    timings = soup.find_all('div', class_='vmXl vmXl-mod-variant-large')
    driver.implicitly_wait(10)

    if len(timings) > 0:
        deptimes = [timing.find_all('span')[0].text.strip() for timing in timings]
        arrtimes = [timing.find_all('span')[2].text.strip() for timing in timings]
    
        deptime = np.array(deptimes)
        arrtime = np.array(arrtimes)

        if deptime.size == 1:
            # Handle case when deptime has only one element
            deptime = np.array([deptime[0], ''])
        deptime = deptime.reshape(int(len(deptime)/2), 2)

        if arrtime.size == 1:
            # Handle case when arrtime has only one element
            arrtime = np.array([arrtime[0], ''])
        arrtime = arrtime.reshape(int(len(arrtime)/2), 2)

        meridiem = ''

else:
        print("No timings found on the page")

    #Get the price
    price_list = soup.find_all('div', class_='f8F1-price-text')
    
    price = []
    for div in price_list:
        print(div.getText().split('\n'))
        price.append(int(div.getText()[1:].replace(',', '')))

    flight_list = soup.find_all('div', class_='c_cgF c_cgF-mod-variant-default')

    flight = []
    for div in flight_list:
        flight.append(div.getText())

    stopover_list = soup.find_all('div', class_='c_cgF c_cgF-mod-variant-default')

    stopovers = []
    for div in stopover_list:
        stopover = div.find_all('span', title=True)
        stopovers.append([span.get_text(strip=True) for span in stopover])

    
    data = []
    for i in range(len(origin)):
        row = {
            "origin": origin,
            "destination": destination,
            "startdate": startdate,
            "enddate": enddate,
            "price": price[i],
            "currency": "USD",
            "deptime_o": deptime[i],
            "arrtime_d": arrtime[i],
            "flight": flight[i],
            "stop_overs": stopovers[i]
        }
        data.append(row)

    df = pd.DataFrame(data)
    
    results = pd.concat([results, df], ignore_index=True)

    driver.close() #close the browser

    time.sleep(15) #wait 15sec until the next request
    
    return "success"


# Create an empty DataFrame
results = pd.DataFrame(columns=['origin','destination','startdate','enddate','deptime_o','arrtime_d','currency','price','flight','stop_overs'])

requests = 0 

destinations = ['BWI']
startdates = ['2023-11-01','2023-11-02']

# Loop over destinations and start dates
for destination in destinations:
    for startdate in startdates:
        requests = requests + 1
        while scrape('DXB', destination, startdate, 3, requests) != "success":
            requests = requests + 1

# Aggregating the flight results
results_agg = results.groupby(['destination', 'startdate'])[['flight', 'stop_overs', 'price']].min().reset_index()

# Save results DataFrame to CSV file
results.to_csv('flight_results.csv', index=False)
results_agg.to_csv('flight_results_agg.csv', index=False)

# Send email with results_agg as attachment
email_subject = 'Flight Results'

# Convert results_agg to a string representation
results_agg_text = results_agg.to_string(index=False)

# Create the email body including results_agg as text
email_body = f'Please find the flight results attached.\n\nFlight Results:\n{results_agg_text}'


sender_email = 'you@sample.com'
receiver_email = 'notifiy@mail.com'
password = 'xxxxxxxx'

msg = MIMEMultipart()
msg['From'] = sender_email
msg['To'] = receiver_email
msg['Subject'] = email_subject

msg.attach(MIMEText(email_body, 'plain'))

# Attach results_agg CSV file to the email
attachment_filename = 'flight_results_agg.csv'
attachment = open(attachment_filename, 'wb')
attachment.write(results_agg.to_csv().encode())
attachment.close()

attachment = open(attachment_filename, "rb")

p = MIMEBase('application', 'octet-stream')
p.set_payload((attachment).read())

encoders.encode_base64(p)
p.add_header('Content-Disposition', f'attachment; filename={attachment_filename}')
msg.attach(p)

# Send the email
smtp_server = 'smtp.sample.com'
smtp_port = 587

try:
    server = smtplib.SMTP(smtp_server, smtp_port)
    server.starttls()
    server.login(sender_email, password)
    text = msg.as_string()
    server.sendmail(sender_email, receiver_email, text)
    server.quit()
    print("Email sent successfully")
except Exception as e:
    print("Error sending email:", str(e))
```

## Pandas Screen Output
![image](https://github.com/oladejiafo/flight_price_web_scraping/assets/69392408/f55a3410-f11f-41bc-9922-3e181100b5ba)

## Output in CSV
![image](https://github.com/oladejiafo/flight_price_web_scraping/assets/69392408/76f25148-c6c6-47a6-8f9d-88c9ae1ad10a)

