# ESP8266_Webserver_Graphing
Using Highcharts to graph multiple inputs from ESP8266
ESP8266 Webserver multiple signal serial plotter high refresh

Hello! Today I’m working on adapting This Tutorial to run at a 4Hz refresh rate or faster, reporting two analytes. I’ll be using an ESP8266 with a loose antenna in A0 (just a wire in the A0 with no return) and a digital temperature sensor from Elegoo. 



To start, I’m going to try to adapt it to a prior code I’ve worked on, that got as far as posting text data points to a website at 2Hz refresh rate, but couldn’t capture a full stream because the string length was set to a numerical limit, and not the refresh timer. 

So the first thing I did was edit down the index.html and the Arduino codes to match the number (and type) of signals I’m sending. Here they are:
ESP8266 Webserver multiple signal serial plotter high refresh

Hello! Today I’m working on adapting This Tutorial to run at a 4Hz refresh rate or faster, reporting two analytes. I’ll be using an ESP8266 with a loose antenna in A0 (just a wire in the A0 with no return) and a digital temperature sensor from Elegoo. 



To start, I’m going to try to adapt it to a prior code I’ve worked on, that got as far as posting text data points to a website at 2Hz refresh rate, but couldn’t capture a full stream because the string length was set to a numerical limit, and not the refresh timer. 

So the first thing I did was edit down the index.html and the Arduino codes to match the number (and type) of signals I’m sending. Here they are:
And, without changing the duration of any of the steps from the original, here’s the output of the code:

Serial monitor posts a string of float numbers

And the website graphs two separate charts live!

Now, the 1Hz refresh and sensor draw rate is a bit of a problem for the goal of our monitor, but hey, the signal gets graphed!

Time to see if the whole setup runs at a distance, and then we can polish the setup, including editing the web page title, calibrating the numbers, and speeding up the refresh, or maybe separating the refresh and sample rates, so that we get multiple data points per pull. 

Looks good overall from across the room; there doesn’t look to be any data loss (though I probably want to attach the antenna, so that it’s reading more accurately)

Next to edit the web page and speed up the refresh rate to .2 seconds (5Hz):

And, we’ve run into an unanticipated problem; the Highcharts program can’t keep up with the refresh rate as far as stitching the lines. But, on the other side, the data is coming in clean, and the ESP is having no trouble sending data at 5Hz! Let’s see if we can just shut off the stitching, and speed up the refresh a bit more. So I’ve set the chart type to Column, so that there’s no line to stitch, and sped up the refresh to 10Hz. Hopefully I wrote the Java section correctly. Nope! Nothing plotted. 

After a little bit more checking, I switched back to Dot Plot, and set lineWidth to 0, and it works!

There’s a default “turn on the line” option when you hover, which I did when taking the snip, but both charts are default with the lines off. 
Trying to boost the refresh rate to 100Hz and x axis range for the A0 to 1000 data points to get closer to realtime data led to significant lag, so it’s time to look for something else. 

Next up, seeing if we can separate the sample rate from the refresh rate. I’ll only do this for the A0 antennae, because the sonar distance sensor has a natural delay because it’s waiting for an echo for every sample. I commented the code a bit better, and clarified sections, so that it’ll be easier to edit. Here’s what it looks like:

I’m going to start by trying to induce multiple entries in the current parser using 

      A += currentAntennae;
      A +="/n"

In the void loop for gathering data, and that meant switching the A from a float to a string in the declarations. And it still works, but there aren’t additional data points arriving on the page yet. 
I’ve added an itemDelimiter to the data section for the motion, and let’s see what that does. Doesn’t graph. 

Ok, I think I need to switch tacks for a bit, and think about the structure of the data. I’ve got x and y coordinates for every data point, and I don’t really need the x data points, because the x axis doesn’t need to be signed since the signal is pretty clear. Let me see if I can get rid of the x axis, and maybe that’ll simplify the parsing. Nope. Didn’t know how to define the x coordinates without a successor function of some kind. Maybe I’ll add a step counter so that the x variable at least isn’t pulling data from datetime for every data point. We can do that in a minute, but I should first check if the A string is getting all the way through to the website, or if it’s getting truncated on the way. 

And, looking at this console output, it does look like the website is pulling at close to the sampling rate of the A0. On top of that, since the antennae and the sonar are in the same void loop without any “if” or “while” statements, they’ll be running in tandem, which means we’re not losing very many sonar samples, either. But, we are still probably not operating at the peak possible speed of the A0, so we’ve got a few challenges ahead for increasing the graphable sampling rate for the A0:

Separate the loops for each sensor. I could probably get the sonar to only run when the server pull is called, and leave the A0 in the void loop so it can run at maximum. 
Either chunk the data for transmission and then parse it into multiple datapoints on the webserver side, or setup a websocket for the whole system to remove headers and stream the data without pull requests. 

I think I’ll try to turn the sonar read function into a subprocess of the pull request, so that it runs at exactly the refresh rate of the website, and if that speeds up the A0 sample rate without a websocket, then we can try chunking. If that doesn’t work, then we’ll go to a websocket for both. 
Here’s the adapted code chunk with the Sonar sensor read being queued by the server pull request:


server.on("/distance", HTTP_GET, [](AsyncWebServerRequest *request){
    request->send_P(200, "text/plain", String(D).c_str());
    
    // Read Sonar
    float currentDistance = distance;
    // Clears the trigPin
    digitalWrite(trigPin, LOW);
    delayMicroseconds(2);
    // Sets the trigPin on HIGH state for 10 micro seconds
    digitalWrite(trigPin, HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPin, LOW);
    // Reads the echoPin, returns the sound wave travel time in microseconds
    duration = pulseIn(echoPin, HIGH);
    // Calculating the distance
    distance = duration * 0.034 / 2;
    // if Sonar read failed, we don't want to change D value
    if (isnan(currentDistance)) {
      Serial.println("Failed to read from Sonar!");
      Serial.println(currentDistance);
    } else {
      D = currentDistance;
      Serial.println(D);
    }
  });


The console output for the data when I was actively watching the page showed a column of float numbers. 
And when I switch to another tab, the console piles up with multiple numbers in the same line, rather than a clean column of individual inputs. 

So, clearly, the A0 pull rate wasn’t delayed by the sonar, and the browser is in full control now of the sample read rate of the sonar. In thinking about the goal of the project, I don’t think we’ll need to worry about losing data when switching tabs; this should be a dedicated monitor that’s always watching the sensors while in use. So, for simplicity sake, we’ll move the sonar read function back to the void loop, and look at including a websocket and see if that can speed up the pull rate. 

Before we edit to bring in a websocket, because that can be tricky (I have another exercise that failed; trying to use websockets to connect RStudio directly to the ESP8266), let’s see if there’s a little more we can get out of the refresh rate of the website. We are close to the sample limit of the sonar the way it’s written, but if we squeeze it’s sample rate a bit, maybe we can also squeeze the webpage refresh rate, and get better than 10Hz. 

The sonar burst is set to 10 microseconds, and we haven’t seen it fail a reading yet, so maybe it can get a bit faster if we set it to 5 microseconds. At 5 microseconds it continued reading, but seemed to lose a lot of its long distance accuracy. It couldn’t tell where my hand was from more than 6 inches away. 

At 7 microseconds the reliable read distance goes up to about a foot and a half (which makes sense, considering the speed of sound), but, given those constraints are probably similar to those of a real monitor, let’s go back to 5 microseconds, keep our hand closer than 6 inches for testing reliability, and boost the refresh rate of the page. 

We’re going to 75 millisecond refresh rate, which I think means 13 Hz or so. 
At this rate, I’m starting to see some website performance issues, where the screen will occasionally freeze, and I’ll see the A0 data pile up. It shows up in the console still, which is nice, but it doesn’t get parsed back out to fill in the gap that happened during the freeze. Also, the sonar is still reliable at 6 inches with the 5 millisecond pulse duration, so we can leave that there. 

It’s time to look into parsing built up chunks, though, to see if we can tweak our way out of the website freezing issue. If we can parse an array into multiple points, then we’ll be able to preserve the sample rate of the A0, and make the website more hardy against lag. 

(And, with some research, I’ve found out that the x axis in HighCharts is required to be DateTime format, so we can scratch that off the list of performance boosts.)

Since the data retrieval and parsing is through XML, I’ll need to use a value splitter using XML commands. What I’ve found so far is splitText(), which so far only allows splits at specific intervals, unlike parsing a CSV that reads up until a special character as a separator. This could be a problem, since the numbers can vary in the number of digits transmitted, unless I tell the Arduino to trim each datapoint to a specific number of digits…which would possibly reduce accuracy. 

And, after searching a bit for that, I’ve discovered a Websocket with Highcharts base code for a similar Arduino Wifi chip, that I’ll try instead. So that’s it for this project! We got it to a 13Hz refresh rate, reliably sampling one analog and one digital sensor. So far so good!


