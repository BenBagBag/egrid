# How Green is My Grid?

## The Problem of Dirty Electricity

Climate scientists predict that we need to have net zero carbon emissions by 2050 in order to avoid the worst effects of global warming ["tipping point"](https://www.un.org/en/climatechange/net-zero-coalition#:~:text=To%20keep%20global%20warming%20to,reach%20net%20zero%20by%202050.). The shift away from using fossil fuels to power cars, stoves, and so on is a critical and readily visible part of this effort. However, moving from fossil fuels to electric power can't help us achieve net zero emissions unless electricity production itself has net zero emissions. Many of our power plants use CO2-emitting sources such as coal and natural gas to produce electricity. 

The problem of CO2-heavy electricity production will only grow. The US is estimated to [increase its electricity consumption](https://www.nationalgrid.com/stories/energy-explained/how-will-our-electricity-supply-change-future) by 50% by 2036 and will double by 2050. If we continue to use coal and fossil fuels to produce electricity, our CO2 emissions will be *increasing* at a time when the planet's future depends on our ability to *reduce* emissions.

A note on terminology: The terms "clean energy", "green energy", and "renewable energy" are often used interchangeably. ["Green energy"](https://www.twi-global.com/technical-knowledge/faqs/what-is-green-energy), in theory, does not produce carbon emissions. However, as we will see, some "green" electricity sources, like hydropower, do create carbon emissions. To avoid confusion, we will refer to fuels that produce CO2. We will also discuss fossil/non-fossil fuels, since fossil fuels are generally understood as a pollution source, while non-fossil fuels are erroneously imagined to be emission-free. 


### Questions:

#### Worst fuels
- Which fuels are the biggest contributors to carbon emissions from power plants?
- Which fuel(s), if eliminated from the US power grid, would have the largest impact on CO2 emissions?

#### Increased demand for electricity
- What percent of power plants are currently using fossil fuels? What percent of power plants use only non-fossil fuels?
- How much electricity did these plants produce in 2021? How much power would we expect to need from them by 2036 and 2050?
- Do we have extra capacity in the power grid? I.e., which power plants would be able to produce more power as our demand for energy increases, without any changes to the plants themselves?
- What will the overall impact of increased demand for electricity be in terms of pollution?
- What would the impact of increased demand be if we removed all fossil fuels from the power grid?

## Data Source

The US Environmental Protection Agency (EPA) releases the [eGrid report](https://www.epa.gov/egrid) each year. This report contains data on each of the 11K power plants in the US and Puerto Rico, including power sources, pollution, and efficiency. It also contains a summary of demographic information for the area surrounding each power plant. The most recent data is from 2021.

A full description of all terms and data in the dataset can be found in [this guide](https://www.epa.gov/system/files/documents/2023-01/eGRID2021_technical_guide.pdf).

Federal regulations require power plants to report their emissions and energy use. This is the data that is presented in the eGrid report. The EPA describes the dataset as containing information on "almost all electric power generated in the United States". It's not clear which power plants would be exempt from this reporting rule and what impact that missing data might have on analysis of the dataset. However, eGrid is used throughout the US government and in industry to calculate the environmental impact of power production, so we will follow the consensus that the dataset is representative of the entire country's power production.

## Clean Data

I cleaned the EPA's eGrid data by:
- extracting and renaming the relevant columns
- creating a schema of expected data types to catch irregularities, which led me to:
- casting columns to the correct data types
- removing rows with 'NaN' in critical columns
- normalizing plant owner and utility company names
    
You can find a notebook documenting the full data cleaning process [here](data/clean_egrid_data.ipynb).

## Load Libraries


```python
import pandas as pd
import numpy as np

from bokeh.plotting import figure, show
from bokeh.io import output_notebook
from bokeh.palettes import Category20c, tol, Category10
from bokeh.transform import cumsum
from bokeh.layouts import row, column
from bokeh.models import BoxZoomTool, ResetTool, ColumnDataSource

import math
```

## Load Data


```python
egrid_df = pd.read_csv('data/cleaned_egrid_data.csv')
```


```python
egrid_df.columns
```




    Index(['Unnamed: 0', 'plant_sequence_num', 'state', 'plant_owner',
           'utility_name', 'balancing_auth_code', 'nerc_region', 'egrid_subregion',
           'county', 'latitude', 'longitude', 'primary_fuel',
           'primary_fuel_category', 'capacity_factor', 'nameplate_capacity_mw',
           'annual_net_generation_mwh', 'annual_co2_emissions_tons',
           'annual_co2_equiv_emissions_tons', 'annual_co2_emission_rate_lb/mwh',
           'annual_co2_equiv_emissions_rate_lb_mwh',
           'annual_coal_net_generation_mwh', 'annual_oil_net_generation_mwh',
           'annual_gas_net_generation_mwh', 'annual_nuclear_net_generation_mwh',
           'annual_hydro__net_generation_mwh', 'annual_biomass_net_generation_mwh',
           'annual_wind_net_generation_mwh', 'annual_solar_net_generation_mwh',
           'annual_geothermal_net_generation_mwh',
           'annual_other_fossil_fuel_net_generation_mwh',
           'annual_other_purchased_net_generation_mwh', 'coal_generation_percent',
           'oil_generation_percent', 'gas_generation_percent',
           'nuclear_generation_percent', 'hydro_generation_percent',
           'biomass_generation_percent', 'wind_generation_percent',
           'solar_generation_percent', 'geothermal_generation_percent',
           'other_fossil_fuel_generation_percent',
           'other_purchased_generation_percent'],
          dtype='object')



## What type of fuel does the majority of our power come from?


```python
fuel_production_columns = [
    'annual_coal_net_generation_mwh', 'annual_oil_net_generation_mwh',
   'annual_gas_net_generation_mwh', 'annual_nuclear_net_generation_mwh',
   'annual_hydro__net_generation_mwh', 'annual_biomass_net_generation_mwh',
   'annual_wind_net_generation_mwh', 'annual_solar_net_generation_mwh',
   'annual_geothermal_net_generation_mwh',
   'annual_other_fossil_fuel_net_generation_mwh',
   'annual_other_purchased_net_generation_mwh'
]

fuel_type_names = [
    'Coal',
    'Oil',
    'Gas',
    'Nuclear',
    'Hydro',
    'Biomass',
    'Wind',
    'Solar',
    'Geothermal',
    'Other fossil fuels',
    'Unknown/purchased'
]

fuel_production_amts = [[egrid_df[type].sum()] for type in fuel_production_columns]

fuel_production_by_type = pd.DataFrame(data=dict(zip(fuel_type_names, fuel_production_amts))).transpose()
```


```python
pd.set_option('display.float_format', '{:.2g}'.format)
```


```python
fuel_production_by_type = fuel_production_by_type.sort_values(by=0, ascending=False)
fuel_production_by_type.columns = ['Annual Production (MWh)']
fuel_production_by_type.index.name = 'Fuel Type'

total_power_production = fuel_production_by_type['Annual Production (MWh)'].sum()
fuel_production_by_type['Percent of annual production'] = fuel_production_by_type['Annual Production (MWh)'] / total_power_production

fuel_production_by_type
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Annual Production (MWh)</th>
      <th>Percent of annual production</th>
    </tr>
    <tr>
      <th>Fuel Type</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>0.38</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>0.22</td>
    </tr>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>0.19</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0.092</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>0.06</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0.028</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>0.013</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>0.0063</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>0.0047</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>0.0039</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>0.001</td>
    </tr>
  </tbody>
</table>
</div>




```python
dirty_fuel_types = [
    'Coal',
    'Oil',
    'Gas',
    'Other fossil fuels'
]

clean_fuel_types = [
    'Nuclear',
    'Hydro',
    'Biomass',
    'Wind',
    'Solar',
    'Geothermal'
]

clean_fuel_production = fuel_production_by_type.loc[clean_fuel_types].sum()
print('Electricity from fossil fuels:')
print(clean_fuel_production)
print('\n')

dirty_fuel_production = fuel_production_by_type.loc[dirty_fuel_types].sum()
print('Electricity from non-fossil fuels:')
print(dirty_fuel_production)
```

    Electricity from fossil fuels:
    Annual Production (MWh)        1.6e+09
    Percent of annual production      0.39
    dtype: float64
    
    
    Electricity from non-fossil fuels:
    Annual Production (MWh)        2.5e+09
    Percent of annual production      0.61
    dtype: float64



```python
# set up notebook to display Bokeh charts

output_notebook()
```


<style>
        .bk-notebook-logo {
            display: block;
            width: 20px;
            height: 20px;
            background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABQAAAAUCAYAAACNiR0NAAAABHNCSVQICAgIfAhkiAAAAAlwSFlzAAALEgAACxIB0t1+/AAAABx0RVh0U29mdHdhcmUAQWRvYmUgRmlyZXdvcmtzIENTNui8sowAAAOkSURBVDiNjZRtaJVlGMd/1/08zzln5zjP1LWcU9N0NkN8m2CYjpgQYQXqSs0I84OLIC0hkEKoPtiH3gmKoiJDU7QpLgoLjLIQCpEsNJ1vqUOdO7ppbuec5+V+rj4ctwzd8IIbbi6u+8f1539dt3A78eXC7QizUF7gyV1fD1Yqg4JWz84yffhm0qkFqBogB9rM8tZdtwVsPUhWhGcFJngGeWrPzHm5oaMmkfEg1usvLFyc8jLRqDOMru7AyC8saQr7GG7f5fvDeH7Ej8CM66nIF+8yngt6HWaKh7k49Soy9nXurCi1o3qUbS3zWfrYeQDTB/Qj6kX6Ybhw4B+bOYoLKCC9H3Nu/leUTZ1JdRWkkn2ldcCamzrcf47KKXdAJllSlxAOkRgyHsGC/zRday5Qld9DyoM4/q/rUoy/CXh3jzOu3bHUVZeU+DEn8FInkPBFlu3+nW3Nw0mk6vCDiWg8CeJaxEwuHS3+z5RgY+YBR6V1Z1nxSOfoaPa4LASWxxdNp+VWTk7+4vzaou8v8PN+xo+KY2xsw6une2frhw05CTYOmQvsEhjhWjn0bmXPjpE1+kplmmkP3suftwTubK9Vq22qKmrBhpY4jvd5afdRA3wGjFAgcnTK2s4hY0/GPNIb0nErGMCRxWOOX64Z8RAC4oCXdklmEvcL8o0BfkNK4lUg9HTl+oPlQxdNo3Mg4Nv175e/1LDGzZen30MEjRUtmXSfiTVu1kK8W4txyV6BMKlbgk3lMwYCiusNy9fVfvvwMxv8Ynl6vxoByANLTWplvuj/nF9m2+PDtt1eiHPBr1oIfhCChQMBw6Aw0UulqTKZdfVvfG7VcfIqLG9bcldL/+pdWTLxLUy8Qq38heUIjh4XlzZxzQm19lLFlr8vdQ97rjZVOLf8nclzckbcD4wxXMidpX30sFd37Fv/GtwwhzhxGVAprjbg0gCAEeIgwCZyTV2Z1REEW8O4py0wsjeloKoMr6iCY6dP92H6Vw/oTyICIthibxjm/DfN9lVz8IqtqKYLUXfoKVMVQVVJOElGjrnnUt9T9wbgp8AyYKaGlqingHZU/uG2NTZSVqwHQTWkx9hxjkpWDaCg6Ckj5qebgBVbT3V3NNXMSiWSDdGV3hrtzla7J+duwPOToIg42ChPQOQjspnSlp1V+Gjdged7+8UN5CRAV7a5EdFNwCjEaBR27b3W890TE7g24NAP/mMDXRWrGoFPQI9ls/MWO2dWFAar/xcOIImbbpA3zgAAAABJRU5ErkJggg==);
        }
    </style>
    <div>
        <a href="https://bokeh.org" target="_blank" class="bk-notebook-logo"></a>
        <span id="p1001">Loading BokehJS ...</span>
    </div>






```python
fuel_x_range = list(fuel_production_by_type.index)
fuel_type_chart = figure(x_range=fuel_x_range, height=350, title="Annual Power Production by Fuel Type", toolbar_location=None, tools="")
fuel_type_chart.vbar(x=fuel_x_range, top=list(fuel_production_by_type['Annual Production (MWh)']), width=0.9)

fuel_type_chart.xgrid.grid_line_color = None
fuel_type_chart.y_range.start = 0
fuel_type_chart.xaxis.major_label_orientation = math.pi/4

show(fuel_type_chart)
```



<div id="1dcb951d-04ef-435c-bf7f-75989a2ad809" data-root-id="p1002" style="display: contents;"></div>





This breakdown of power sources shows us that:

- two fossil fuels, gas and coal, produce 60% of our electricity, meaning that we need to replace the majority of our power production in order to reach zero emissions
- nearly 40% of our electricity comes from non-fossil fuel sources, with nuclear and wind as the largest sources of clean energy
- oil amounts for only 0.6% of electricity from power plants. This is unsurprising given that electricity production accounts for only only [0.5% of US oil use](https://www.eia.gov/energyexplained/oil-and-petroleum-products/use-of-oil.php); oil is used mostly for [heating](https://www.eia.gov/energyexplained/heating-oil/use-of-heating-oil.php#:~:text=Who%20uses%20heating%20oil%3F,the%20U.S.%20Northeast%20Census%20Region.) (although this use is on the decline)
- solar power is only 2.8% of electricity produced in power plants in the US. This is 70% of [*all solar electricity* in the US](https://www.eia.gov/energyexplained/solar/where-solar-is-found.php). This suggests that solar electricity is severely underutilized
- geothermal energy accounts for only 3.9% of our electricity. However, the US is the [largest producer of geothermal energy](https://www.eia.gov/energyexplained/geothermal/use-of-geothermal-energy.php) 

## What are the greenest fuel types?

Moving towards a cleaner power grid will require choices about when to phase out existing power plants that produce CO2 emissions. To eliminate CO2 as quickly as possible, we should focus on the worst polluters. 

Which dirty fuels produce the most CO2? Are there differences between non-fossil fuels?

We will restrict this analysis to power plants that use only one fuel type; in plants using more than one fuel type, there is not enough data to determine what percent of the CO2 produced is from each fuel source.

### Identify single-fuel power plants


```python
fuel_percent_cols = [
'coal_generation_percent',
'oil_generation_percent',
'gas_generation_percent',
'nuclear_generation_percent', 
'hydro_generation_percent',
'biomass_generation_percent', 
'wind_generation_percent',
'solar_generation_percent', 
'geothermal_generation_percent',
'other_fossil_fuel_generation_percent',
'other_purchased_generation_percent'
]

egrid_df['is_single_fuel_plant'] = egrid_df.apply(lambda x: x[fuel_percent_cols].transpose().gt(0).sum() == 1, axis=1)
```


```python
egrid_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Unnamed: 0</th>
      <th>plant_sequence_num</th>
      <th>state</th>
      <th>plant_owner</th>
      <th>utility_name</th>
      <th>balancing_auth_code</th>
      <th>nerc_region</th>
      <th>egrid_subregion</th>
      <th>county</th>
      <th>latitude</th>
      <th>...</th>
      <th>gas_generation_percent</th>
      <th>nuclear_generation_percent</th>
      <th>hydro_generation_percent</th>
      <th>biomass_generation_percent</th>
      <th>wind_generation_percent</th>
      <th>solar_generation_percent</th>
      <th>geothermal_generation_percent</th>
      <th>other_fossil_fuel_generation_percent</th>
      <th>other_purchased_generation_percent</th>
      <th>is_single_fuel_plant</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>3</td>
      <td>3</td>
      <td>AK</td>
      <td>Copper Valley Elec Assn, Inc</td>
      <td>Copper Valley Elec Assn, Inc</td>
      <td>UNKNOWN</td>
      <td>AK</td>
      <td>AKMS</td>
      <td>Valdez Cordova</td>
      <td>61</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4</td>
      <td>4</td>
      <td>AK</td>
      <td>Alaska Village Elec Coop, Inc</td>
      <td>Alaska Village Elec Coop, Inc</td>
      <td>UNKNOWN</td>
      <td>AK</td>
      <td>AKMS</td>
      <td>Northwest Arctic</td>
      <td>67</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>5</td>
      <td>5</td>
      <td>AK</td>
      <td>Inside Passage Elec Coop, Inc</td>
      <td>Inside Passage Elec Coop, Inc</td>
      <td>UNKNOWN</td>
      <td>AK</td>
      <td>AKMS</td>
      <td>Hoonah-Angoon</td>
      <td>57</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>6</td>
      <td>6</td>
      <td>AK</td>
      <td>Aniak Light &amp; Power Co Inc</td>
      <td>Aniak Light &amp; Power Co Inc</td>
      <td>UNKNOWN</td>
      <td>AK</td>
      <td>AKMS</td>
      <td>Bethel</td>
      <td>62</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>7</td>
      <td>7</td>
      <td>AK</td>
      <td>Alaska Electric Light&amp;Power Co</td>
      <td>Alaska Electric Light &amp; Power Co.</td>
      <td>UNKNOWN</td>
      <td>AK</td>
      <td>AKMS</td>
      <td>Juneau</td>
      <td>58</td>
      <td>...</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 43 columns</p>
</div>




```python
single_fuel_plants_df = egrid_df[egrid_df['is_single_fuel_plant'] == True]
```

### Which fuels are the least efficient?

The `annual_co2_emission_rate_lb/mwh` column can be used to determine which fuels produce the most CO2 emissions relative to the electricity they produce. The ratio of CO2 emissions : power production (in MWh of electricity) shows us the environmental impact of different fuels. (As a point of comparison, one MWh of electricity can power [two fridges for a year](https://www.freeingenergy.com/what-is-a-megawatt-hour-of-electricity-and-what-can-you-do-with-it/)).

Findings:

Oil, gas, and coal are the biggest polluters when measured by pounds of CO2 produced per MWh of electricity. 

The cleanest fuels are wind, solar, and nuclear. We also see that geothermal power produces nearly 25 times more CO2 than biomass, the next cleanest fuel on the list. This is in keeping with the debate over whether or not geothermal power is really "green"; it produces much more CO2 than other green fuels in addition to the water and air pollution.


```python
fuel_type_efficiency = single_fuel_plants_df.groupby('primary_fuel_category')['annual_co2_emission_rate_lb/mwh'].mean().sort_values(ascending=False)
fuel_type_efficiency.index.name = 'Fuel type'
fuel_type_efficiency = fuel_type_efficiency.reset_index().rename(columns={ 'annual_co2_emission_rate_lb/mwh': 'CO2 emissions lb/MWh' })
```


```python
# Add a column for metric tons/MWh (this is a more common measure of CO2 emissions)
lbs_to_metric_tons_factor = 0.000453592
fuel_type_efficiency['CO2 emissions metric tons/MWh'] = fuel_type_efficiency['CO2 emissions lb/MWh'] * lbs_to_metric_tons_factor

# Ignoring the "grab bag" categories, because those include multiple fuel types
fuel_type_efficiency = fuel_type_efficiency[~fuel_type_efficiency['Fuel type'].isin(['OTHF', 'OFSL'])]

fuel_type_efficiency
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fuel type</th>
      <th>CO2 emissions lb/MWh</th>
      <th>CO2 emissions metric tons/MWh</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>OIL</td>
      <td>4.1e+03</td>
      <td>1.9</td>
    </tr>
    <tr>
      <th>1</th>
      <td>GAS</td>
      <td>3.5e+03</td>
      <td>1.6</td>
    </tr>
    <tr>
      <th>2</th>
      <td>COAL</td>
      <td>1.6e+03</td>
      <td>0.75</td>
    </tr>
    <tr>
      <th>4</th>
      <td>GEOTHERMAL</td>
      <td>1e+02</td>
      <td>0.048</td>
    </tr>
    <tr>
      <th>6</th>
      <td>BIOMASS</td>
      <td>4.4</td>
      <td>0.002</td>
    </tr>
    <tr>
      <th>7</th>
      <td>HYDRO</td>
      <td>0.007</td>
      <td>3.2e-06</td>
    </tr>
    <tr>
      <th>8</th>
      <td>NUCLEAR</td>
      <td>0.0032</td>
      <td>1.5e-06</td>
    </tr>
    <tr>
      <th>9</th>
      <td>SOLAR</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>WIND</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>




```python
fuel_type_efficiency.to_csv('co2_emissions_by_fuel_type.csv')
```


```python
fuel_efficiency_graph_df = fuel_type_efficiency[~fuel_type_efficiency['Fuel type'].isin(['OTHF', 'OFSL'])]

fuel_efficiency_x_range = list(fuel_efficiency_graph_df['Fuel type'])

fuel_efficiency_type_chart = figure(x_range=fuel_efficiency_x_range, height=350, title="CO2 Emissions in Metric Tons per MWh of Electricity Produced", toolbar_location='below', tools="hover,wheel_zoom,box_zoom,reset", tooltips="@x: @top metric tons CO2/MWh")
fuel_efficiency_type_chart.vbar(x=fuel_efficiency_x_range, top=list(fuel_efficiency_graph_df['CO2 emissions metric tons/MWh']), width=0.9)

fuel_efficiency_type_chart.xgrid.grid_line_color = None
fuel_efficiency_type_chart.y_range.start = 0
fuel_efficiency_type_chart.xaxis.major_label_orientation = math.pi/4

show(fuel_efficiency_type_chart)
```



<div id="18611262-1920-4140-b09f-e33b7629a770" data-root-id="p523399" style="display: contents;"></div>





**NB: use the magnifying glass icons below the chart to zoom in and out. Reset the chart with the recycle icon**

The chart above shows that oil emits the most CO2 per MWh of electricity produced, followed by gas and coal. There is a dramatic drop between coal and the "clean" fuels: biomass, hydro, nuclear, solar, and wind energy. 

A note on gas and coal: when natural gas is burned in modern, fuel-efficient power plants, it produces about [50% less CO2 than coal](https://www.gasvessel.eu/news/natural-gas-vs-coal-impact-on-the-environment/#:~:text=Natural%20gas%20is%20a%20fossil,a%20typical%20new%20coal%20plant.). However, our data shows **coal** producing 50% less CO2 than gas. This is likely because our data represents actual power plants in the US and, which are not necessarily using the most efficient power production methods. This suggests two important points:
1. Switching gas power plants to more efficient power production methods could be a relatively easy way to reduce emissions from gas. Further research is needed to determine if this is practical, and it doesn't obviate the need to eliminate gas from the power grid.
2. An analysis of which power plants produce the highest CO2 emissions per MWh could provide a "worst offenders" list of power plants that should be replaced as soon as possible to make the largest impact on CO2 emissions. 

If we look at what percent of CO2 output each fuel type is responsible for, will we get a similarly dramatic drop?

### Which fuel types produce the most CO2 emissions?
 
We will find that **94%** of all emissions from power plants are produced by gas.


```python
total_co2_production = single_fuel_plants_df['annual_co2_emissions_tons'].sum()

percent_of_net_emissions_by_fuel = single_fuel_plants_df.groupby('primary_fuel_category')['annual_co2_emissions_tons'].sum().sort_values(ascending=False)
percent_of_net_emissions_by_fuel.index.name = 'Fuel type'
percent_of_net_emissions_by_fuel = percent_of_net_emissions_by_fuel.reset_index().rename(columns={ 'annual_co2_emissions_tons': 'Annual CO2 emissions (tons)' })
percent_of_net_emissions_by_fuel['Percent of annual CO2 emissions'] = percent_of_net_emissions_by_fuel['Annual CO2 emissions (tons)'] / total_co2_production

net_power_by_fuel_type = single_fuel_plants_df.groupby('primary_fuel_category')['annual_net_generation_mwh'].sum()
net_power_by_fuel_type.index.name = 'Fuel type'
net_power_by_fuel_type = net_power_by_fuel_type.reset_index().rename(columns={ 'annual_net_generation_mwh': 'Annual power production MWh' })

percentage_emissions_and_production = pd.merge(percent_of_net_emissions_by_fuel, net_power_by_fuel_type, how='left', on=['Fuel type'])
percentage_emissions_and_production['Percent of annual power production'] = percentage_emissions_and_production['Annual power production MWh'] / percentage_emissions_and_production['Annual power production MWh'].sum()


# Ignoring the "grab bag" categories, because those include multiple fuel types
percentage_emissions_and_production = percentage_emissions_and_production[~percentage_emissions_and_production['Fuel type'].isin(['OTHF', 'OFSL'])]
percentage_emissions_and_production
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fuel type</th>
      <th>Annual CO2 emissions (tons)</th>
      <th>Percent of annual CO2 emissions</th>
      <th>Annual power production MWh</th>
      <th>Percent of annual power production</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>GAS</td>
      <td>4.8e+08</td>
      <td>0.95</td>
      <td>1.1e+09</td>
      <td>0.41</td>
    </tr>
    <tr>
      <th>1</th>
      <td>COAL</td>
      <td>1.3e+07</td>
      <td>0.026</td>
      <td>1.2e+07</td>
      <td>0.0045</td>
    </tr>
    <tr>
      <th>2</th>
      <td>OIL</td>
      <td>1.3e+07</td>
      <td>0.025</td>
      <td>1.4e+07</td>
      <td>0.0053</td>
    </tr>
    <tr>
      <th>3</th>
      <td>GEOTHERMAL</td>
      <td>1.3e+06</td>
      <td>0.0026</td>
      <td>1.6e+07</td>
      <td>0.0059</td>
    </tr>
    <tr>
      <th>5</th>
      <td>BIOMASS</td>
      <td>7.7e+04</td>
      <td>0.00015</td>
      <td>1.4e+07</td>
      <td>0.0054</td>
    </tr>
    <tr>
      <th>7</th>
      <td>NUCLEAR</td>
      <td>8.3e+02</td>
      <td>1.6e-06</td>
      <td>7.5e+08</td>
      <td>0.28</td>
    </tr>
    <tr>
      <th>8</th>
      <td>HYDRO</td>
      <td>4.9e+02</td>
      <td>9.6e-07</td>
      <td>2.5e+08</td>
      <td>0.095</td>
    </tr>
    <tr>
      <th>9</th>
      <td>SOLAR</td>
      <td>0</td>
      <td>0</td>
      <td>1.1e+08</td>
      <td>0.043</td>
    </tr>
    <tr>
      <th>10</th>
      <td>WIND</td>
      <td>0</td>
      <td>0</td>
      <td>3.8e+08</td>
      <td>0.14</td>
    </tr>
  </tbody>
</table>
</div>




```python
fuel_emissions_x_range = list(percentage_emissions_and_production['Fuel type'])

fuel_emissions_type_chart = figure(x_range=fuel_emissions_x_range, height=350, title="Annual Power Plant CO2 Emissions by Fuel Type", toolbar_location='below', tools="hover,wheel_zoom,box_zoom,reset", tooltips="@x: @top metric tons CO2/yr")
fuel_emissions_type_chart.vbar(x=fuel_emissions_x_range, top=list(percentage_emissions_and_production['Annual CO2 emissions (tons)']), width=0.9)

fuel_emissions_type_chart.xgrid.grid_line_color = None
fuel_emissions_type_chart.y_range.start = 10**-8
fuel_emissions_type_chart.xaxis.major_label_orientation = math.pi/4

show(fuel_emissions_type_chart)
```



<div id="8c443fd1-935e-4873-9603-3ab4f62c1de4" data-root-id="p531520" style="display: contents;"></div>





When looking at what percent of fuel-related CO2 emissions each fuel is responsible for, we again see a steep drop-off from gas, coal, and oil emissions and the other, greener fuels. The difference is so dramatic that biomass, nuclear, and hydro fuel (solar and wind produce no CO2) emissions are only visible if the user zooms in. The same is true with a pie chart.

(NB: the EPA estimates that solar and wind power do not produce any CO2 emissions. [This does not include](https://www.epa.gov/system/files/documents/2023-01/eGRID2021_technical_guide.pdf) any emissions created by solar/wind equipment installation, emissions from other plant equipment, etc.)


```python
emissions_pie_data = percentage_emissions_and_production[['Fuel type', 'Annual CO2 emissions (tons)', 'Percent of annual CO2 emissions']]
emissions_pie_data.columns = ['type', 'emissions', 'percent']
emissions_pie_data['percent'] = emissions_pie_data['percent'] * 100
emissions_pie_data['percent'] = emissions_pie_data['percent'].astype('int')

emissions_pie_data['angle'] = emissions_pie_data['emissions'] / emissions_pie_data['emissions'].sum() * 2 * math.pi
emissions_pie_data['color'] = Category20c[len(emissions_pie_data)]

emissions_pie = figure(height=400, title="Annual emissions by fuel type", toolbar_location='below', tools="hover,wheel_zoom,box_zoom,reset", tooltips="@type: @emissions metric tons (@percent %)", x_range=(-0.5, 1.0))

emissions_pie.wedge(x=0, y=1, radius=0.4,
        start_angle=cumsum('angle', include_zero=True), end_angle=cumsum('angle'),
        line_color="white", fill_color='color', legend_field='type', source=emissions_pie_data)

emissions_pie.axis.axis_label = None
emissions_pie.axis.visible = False
emissions_pie.grid.grid_line_color = None

show(emissions_pie)
```

    /var/folders/dv/sn1_d0xn0vncjbncdpkkg_pc0000gn/T/ipykernel_58560/257064240.py:3: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      emissions_pie_data['percent'] = emissions_pie_data['percent'] * 100
    /var/folders/dv/sn1_d0xn0vncjbncdpkkg_pc0000gn/T/ipykernel_58560/257064240.py:4: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      emissions_pie_data['percent'] = emissions_pie_data['percent'].astype('int')
    /var/folders/dv/sn1_d0xn0vncjbncdpkkg_pc0000gn/T/ipykernel_58560/257064240.py:6: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      emissions_pie_data['angle'] = emissions_pie_data['emissions'] / emissions_pie_data['emissions'].sum() * 2 * math.pi
    /var/folders/dv/sn1_d0xn0vncjbncdpkkg_pc0000gn/T/ipykernel_58560/257064240.py:7: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame.
    Try using .loc[row_indexer,col_indexer] = value instead
    
    See the caveats in the documentation: https://pandas.pydata.org/pandas-docs/stable/user_guide/indexing.html#returning-a-view-versus-a-copy
      emissions_pie_data['color'] = Category20c[len(emissions_pie_data)]




<div id="ad53f57b-79be-4e4c-8cb8-e042a565ce5c" data-root-id="p572385" style="display: contents;"></div>






```python
percentage_emissions_and_production.to_csv('compare_heights_emissions.csv')
```

## Current Electricity Production and Future Demand Estimate

We will use the total current electricity production from the dataset's power plants as an estimate of the current electricity produced in the US.

First, add a column to represent each plant's total annual MWh production. A megawatt hour is 1,000 kilowatts of electricity generated per hour. For reference, a megawatt hour of power can keep [two refrigerators running for a year](https://www.freeingenergy.com/what-is-a-megawatt-hour-of-electricity-and-what-can-you-do-with-it/).


```python
annual_power_production = egrid_df['annual_net_generation_mwh'].sum()

print('The total annual production is {total_energy} MWh.'.format(total_energy='{:.2e}'.format(annual_power_production)))
```

    The total annual production is 4.12e+09 MWh.


At a minimum, we can expect electricity consumption to [rise](https://www.nationalgrid.com/stories/energy-explained/how-will-our-electricity-supply-change-future) by 50% by 2036 and to double by 2050. Based on current production, we can estimate that the US will need to produce:


```python
est_2036_need = annual_power_production * 1.5
est_2050_need = annual_power_production * 2

print('The US and Puerto Rico will need {est_2036} MWh/yr by 2036 and {est_2050} MWh/yr by 2050.'.format(est_2036='{:.2e}'.format(est_2036_need), est_2050='{:.2e}'.format(est_2050_need)))
```

    The US and Puerto Rico will need 6.18e+09 MWh/yr by 2036 and 8.24e+09 MWh/yr by 2050.



```python
shortfall_2036 = est_2036_need - annual_power_production
shortfall_2050 = est_2050_need - annual_power_production

print('We will need to produce an additional {shortfall_2036} MWh/yr by 2036 and an additional {shortfall_2050} MWh/yr by 2050 to keep up with demand.'.format(shortfall_2036='{:.2e}'.format(shortfall_2036), shortfall_2050='{:.2e}'.format(shortfall_2050)))
```

    We will need to produce an additional 2.06e+09 MWh/yr by 2036 and an additional 4.12e+09 MWh/yr by 2050 to keep up with demand.


### Estimating additional electric production capacity

An obvious question is to ask if the current power plants are operating at maximum capacity, and if not, how much more energy we could produce from them.

Unfortunately, I was unable to find sufficient data to answer this question. 

The EPA data includes the capacity factor (a ratio of energy produced to potential energy that could be produced) for each plant, and the national average capacity for power plants based on fuel type. 

The difference between the power that is actually produced and the potential power is due to several factors:
    
- downtime for maintenance
- downtime due to lack of fuel (gas shortages, cloudy days for solar plants, etc.)
- downtime or reduced production because the grid has sufficient power
    
Without data to differentiate these three reasons, it's impossible to calculate which plants could actually produce more power day in and day out, year after year. So for the purpose of this analysis, we will assume that our current grid is at capacity. 

## Environmental impact of increased energy needs

What will the environmental impact of increased energy use be?

We will combine two pieces of data:

- the breakdown of US power production by fuel type, based on all power grids in the US
- the average CO2/MWh emissions by fuel type, based on emissions from single-fuel power plants

We'll calculate emissions in three scenarios:

- assuming that the grid will continue to use the same ratio of fossil and non-fossil fuels for increased electricity needs
- assuming any increased electricity production will be from non-fossil fuels
- assuming that we eliminate fossil fuel use for electricity production by 2050 and meet all US electricity needs with non-fossil fuels

A quick refresher of our numbers for 2021:


```python
fuel_production_by_type
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Annual Production (MWh)</th>
      <th>Percent of annual production</th>
    </tr>
    <tr>
      <th>Fuel Type</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>0.38</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>0.22</td>
    </tr>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>0.19</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0.092</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>0.06</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0.028</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>0.013</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>0.0063</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>0.0047</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>0.0039</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>0.001</td>
    </tr>
  </tbody>
</table>
</div>




```python
fuel_type_mappings = { 'OIL': 'Oil', 'GAS': 'Gas', 'COAL': 'Coal', 'OFSL': 'Unknown/purchased', 'OTHF': 'Other fossil fuels', 'GEOTHERMAL': 'Geothermal', 'BIOMASS': 'Biomass', 'HYDRO': 'Hydro', 'NUCLEAR': 'Nuclear', 'SOLAR': 'Solar', 'WIND': 'Wind' }
fuel_type_efficiency = fuel_type_efficiency.replace({'Fuel type': fuel_type_mappings})
```


```python
fuel_type_efficiency = fuel_type_efficiency.set_index('Fuel type')
fuel_type_efficiency
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>CO2 emissions lb/MWh</th>
      <th>CO2 emissions metric tons/MWh</th>
    </tr>
    <tr>
      <th>Fuel type</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Oil</th>
      <td>4.1e+03</td>
      <td>1.9</td>
    </tr>
    <tr>
      <th>Gas</th>
      <td>3.5e+03</td>
      <td>1.6</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>1.6e+03</td>
      <td>0.75</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>5.2e+02</td>
      <td>0.24</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1e+02</td>
      <td>0.048</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>82</td>
      <td>0.037</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>4.4</td>
      <td>0.002</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>0.007</td>
      <td>3.2e-06</td>
    </tr>
    <tr>
      <th>Nuclear</th>
      <td>0.0032</td>
      <td>1.5e-06</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



### Emissions growth without changes in the electric grid makeup


```python
# 2021 
grid_growth_df_no_greening = fuel_production_by_type[['Annual Production (MWh)']]
grid_growth_df_no_greening.columns = ['2021 Annual Production (MWh)']

grid_growth_df_no_greening['2021 CO2 Emissions (metric tons)'] = fuel_type_efficiency['CO2 emissions metric tons/MWh'] * fuel_production_by_type['Annual Production (MWh)']

# 2036
# power needs will rise by 50% from 2021
grid_growth_df_no_greening['2036 Annual Production (MWh)'] = grid_growth_df_no_greening['2021 Annual Production (MWh)'] * 1.5
grid_growth_df_no_greening['2036 CO2 Emissions (metric tons)'] = fuel_type_efficiency['CO2 emissions metric tons/MWh'] * grid_growth_df_no_greening['2036 Annual Production (MWh)']

# 2050
# power needs will rise by 100% from 2021
grid_growth_df_no_greening['2050 Annual Production (MWh)'] = grid_growth_df_no_greening['2021 Annual Production (MWh)'] * 2
grid_growth_df_no_greening['2050 CO2 Emissions (metric tons)'] = fuel_type_efficiency['CO2 emissions metric tons/MWh'] * grid_growth_df_no_greening['2050 Annual Production (MWh)']

grid_growth_df_no_greening
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 Annual Production (MWh)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 Annual Production (MWh)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
    <tr>
      <th>Fuel Type</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>2.4e+09</td>
      <td>3.7e+09</td>
      <td>3.2e+09</td>
      <td>5e+09</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>1.4e+09</td>
      <td>1e+09</td>
      <td>1.8e+09</td>
      <td>1.3e+09</td>
    </tr>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>1.1e+03</td>
      <td>1.2e+09</td>
      <td>1.7e+03</td>
      <td>1.6e+09</td>
      <td>2.3e+03</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0</td>
      <td>5.7e+08</td>
      <td>0</td>
      <td>7.6e+08</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>7.8e+02</td>
      <td>3.7e+08</td>
      <td>1.2e+03</td>
      <td>4.9e+08</td>
      <td>1.6e+03</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0</td>
      <td>1.7e+08</td>
      <td>0</td>
      <td>2.3e+08</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>1.1e+05</td>
      <td>8e+07</td>
      <td>1.6e+05</td>
      <td>1.1e+08</td>
      <td>2.2e+05</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>3.9e+07</td>
      <td>7.3e+07</td>
      <td>5.2e+07</td>
      <td>9.8e+07</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>2.9e+07</td>
      <td>1.1e+06</td>
      <td>3.9e+07</td>
      <td>1.4e+06</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>7.6e+05</td>
      <td>2.4e+07</td>
      <td>1.1e+06</td>
      <td>3.2e+07</td>
      <td>1.5e+06</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>6.3e+06</td>
      <td>1.5e+06</td>
      <td>8.4e+06</td>
      <td>2e+06</td>
    </tr>
  </tbody>
</table>
</div>




```python
fuel_stack_order = ['Gas', 'Coal', 'Oil', 'Other fossil fuels', 'Unknown/purchased', 'Geothermal', 'Nuclear', 'Biomass', 'Hydro', 'Wind', 'Solar']

def make_emissions_stack_chart(source_df, y_min, y_max, title, is_detail):
    emissions_chart = source_df[['2021 CO2 Emissions (metric tons)', '2036 CO2 Emissions (metric tons)', '2050 CO2 Emissions (metric tons)']]
    emissions_chart.columns = ['2021', '2036', '2050']
    emissions_chart = emissions_chart.reindex(fuel_stack_order)
    emissions_chart = emissions_chart.T
    
    if is_detail:
        emissions_chart = emissions_chart[['Geothermal', 'Nuclear', 'Biomass', 'Hydro', 'Wind', 'Solar']]

    emissions_names = list(emissions_chart.columns)
    emissions_chart_fig = figure(height=600, width=800, x_range=(2021, 2050), y_range=(y_min, y_max), x_axis_label='Year', y_axis_label='CO2 Emissions (metric tons)', title=title)
    emissions_chart_fig.grid.minor_grid_line_color = '#eeeeee'

    color_palatte = Category20c[len(fuel_stack_order)]
    color_subset = color_palatte[-len(emissions_chart.columns):] if is_detail else color_palatte
    emissions_chart_fig.varea_stack(stackers=emissions_names, x='index', color=color_subset, legend_label=emissions_names, source=emissions_chart)

    emissions_chart_fig.legend.orientation = 'vertical'
    emissions_chart_fig.legend.background_fill_color = '#fafafa'
    emissions_chart_fig.add_layout(emissions_chart_fig.legend[0], 'right')
    
    return emissions_chart_fig

no_green_emissions_chart_fig = make_emissions_stack_chart(grid_growth_df_no_greening, 0, 6.5*10**9, 'Expected Annual CO2 Emissions Using the Current Fuel Mix', False)
no_green_emissions_chart_fig_detail = make_emissions_stack_chart(grid_growth_df_no_greening, 0, 2*10**6, 'Detail of Green Fuels in Expected Annual CO2 Emissions Using the Current Fuel Mix', True)

show(column(no_green_emissions_chart_fig, no_green_emissions_chart_fig_detail))
```



<div id="34588fdd-7237-4468-a022-07aec040f45b" data-root-id="p416518" style="display: contents;"></div>





### Emissions growth if all increased electricity production is from non-fossil fuels

Assumptions:
- We will keep the same ratio of non-fossil fuel types as 2021 production
- Current power plants will remain in use
- Emissions/MWh for each type of fuel will remain the same (they will likely become more efficient, but we don't have reliable data to make an estimate)


```python
# power and emissions from non-fossil fuels will be the same 2021-2050

dirty_fuels_no_growth_df = grid_growth_df_no_greening.loc[['Gas', 'Coal', 'Oil', 'Other fossil fuels', 'Unknown/purchased']][['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]

dirty_fuels_no_growth_df[['2036 Annual Production (MWh)', '2036 CO2 Emissions (metric tons)']] = dirty_fuels_no_growth_df[['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]
dirty_fuels_no_growth_df[['2050 Annual Production (MWh)', '2050 CO2 Emissions (metric tons)']] = dirty_fuels_no_growth_df[['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]

dirty_fuels_no_growth_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 Annual Production (MWh)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 Annual Production (MWh)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
    <tr>
      <th>Fuel Type</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>9e+08</td>
      <td>6.7e+08</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
    </tr>
  </tbody>
</table>
</div>




```python
# calculate what percent of non-fossil fuel electricity is produced by each fuel type

percent_of_clean_fuel_df = grid_growth_df_no_greening.loc[['Nuclear', 'Wind', 'Hydro', 'Solar', 'Biomass', 'Geothermal']][['2021 Annual Production (MWh)']]
percent_of_clean_fuel_df['Percent of Clean Fuel'] = percent_of_clean_fuel_df['2021 Annual Production (MWh)'] / percent_of_clean_fuel_df['2021 Annual Production (MWh)'].sum()

percent_of_clean_fuel_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>Percent of Clean Fuel</th>
    </tr>
    <tr>
      <th>Fuel Type</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>0.49</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0.24</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>0.16</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0.072</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>0.034</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>0.01</td>
    </tr>
  </tbody>
</table>
</div>




```python
# amount of power from non-fossil fuels in 2036 and 2050

clean_fuels_growth_df = grid_growth_df_no_greening.loc[['Nuclear', 'Wind', 'Hydro', 'Solar', 'Biomass', 'Geothermal']][['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]

dirty_fuel_production = dirty_fuels_no_growth_df['2021 Annual Production (MWh)'].sum()
```


```python
clean_fuel_production_2036 = grid_growth_df_no_greening['2036 Annual Production (MWh)'].sum() - dirty_fuel_production
clean_fuel_production_2050 = grid_growth_df_no_greening['2050 Annual Production (MWh)'].sum() - dirty_fuel_production
```


```python
clean_fuels_growth_df['2036 Annual Production (MWh)'] = percent_of_clean_fuel_df['Percent of Clean Fuel'] * clean_fuel_production_2036
clean_fuels_growth_df['2036 CO2 Emissions (metric tons)'] = clean_fuels_growth_df['2036 Annual Production (MWh)'] * fuel_type_efficiency['CO2 emissions metric tons/MWh']

clean_fuels_growth_df['2050 Annual Production (MWh)'] = percent_of_clean_fuel_df['Percent of Clean Fuel'] * clean_fuel_production_2050
clean_fuels_growth_df['2050 CO2 Emissions (metric tons)'] = clean_fuels_growth_df['2050 Annual Production (MWh)'] * fuel_type_efficiency['CO2 emissions metric tons/MWh']

clean_fuels_growth_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 Annual Production (MWh)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 Annual Production (MWh)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
    <tr>
      <th>fuel</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>1.1e+03</td>
      <td>1.8e+09</td>
      <td>2.6e+03</td>
      <td>2.8e+09</td>
      <td>4.1e+03</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0</td>
      <td>8.7e+08</td>
      <td>0</td>
      <td>1.4e+09</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>7.8e+02</td>
      <td>5.7e+08</td>
      <td>1.8e+03</td>
      <td>8.9e+08</td>
      <td>2.8e+03</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0</td>
      <td>2.6e+08</td>
      <td>0</td>
      <td>4.1e+08</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>1.1e+05</td>
      <td>1.2e+08</td>
      <td>2.5e+05</td>
      <td>1.9e+08</td>
      <td>3.9e+05</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>7.6e+05</td>
      <td>3.7e+07</td>
      <td>1.7e+06</td>
      <td>5.7e+07</td>
      <td>2.7e+06</td>
    </tr>
  </tbody>
</table>
</div>




```python
grid_growth_df_with_greening = pd.concat([clean_fuels_growth_df, dirty_fuels_no_growth_df])

grid_growth_df_with_greening
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 Annual Production (MWh)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 Annual Production (MWh)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>1.1e+03</td>
      <td>1.8e+09</td>
      <td>2.6e+03</td>
      <td>2.8e+09</td>
      <td>4.1e+03</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0</td>
      <td>8.7e+08</td>
      <td>0</td>
      <td>1.4e+09</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>7.8e+02</td>
      <td>5.7e+08</td>
      <td>1.8e+03</td>
      <td>8.9e+08</td>
      <td>2.8e+03</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0</td>
      <td>2.6e+08</td>
      <td>0</td>
      <td>4.1e+08</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>1.1e+05</td>
      <td>1.2e+08</td>
      <td>2.5e+05</td>
      <td>1.9e+08</td>
      <td>3.9e+05</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>7.6e+05</td>
      <td>3.7e+07</td>
      <td>1.7e+06</td>
      <td>5.7e+07</td>
      <td>2.7e+06</td>
    </tr>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>9e+08</td>
      <td>6.7e+08</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
    </tr>
  </tbody>
</table>
</div>




```python
green_emissions_chart = make_emissions_stack_chart(grid_growth_df_with_greening, 0, 3.4 * 10**9, 'Expected CO2 Emissions Using Only Clean Fuels for Increased Power Needs', False)
green_emissions_chart_detail = make_emissions_stack_chart(grid_growth_df_with_greening, 0, 3.2 * 10**6, 'Detail of Expected CO2 Emissions Using Only Clean Fuels for Increased Power Needs', True)

show(column(green_emissions_chart, green_emissions_chart_detail))
```



<div id="ffb678a2-9fcb-4b17-b07c-5343016e5e09" data-root-id="p724032" style="display: contents;"></div>





## Emissions growth if fossil fuels are removed from the power grid

Our final model will look at emissions if the US power grid stops using fossil fuels.

Model:
 - Emissions from fossil fuels are reduced by 50% in 2036 and eliminated entirely in 2050
 - Both increased electricity demands and the electricity needed to replace what was produced by fossil fuels will be produced with non-fossil fuels, using the same ratio of production per fuel type as in 2021


```python
fossil_fuel_reduction_df = grid_growth_df_no_greening.loc[['Gas', 'Coal', 'Oil', 'Other fossil fuels', 'Unknown/purchased']][['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]

fossil_fuel_reduction_df['2036 CO2 Emissions (metric tons)'] = fossil_fuel_reduction_df['2021 CO2 Emissions (metric tons)'] * 0.5
fossil_fuel_reduction_df['2036 Annual Production (MWh)'] = fossil_fuel_reduction_df['2036 CO2 Emissions (metric tons)'] / fuel_type_efficiency['CO2 emissions metric tons/MWh']

fossil_fuel_reduction_df.columns = ['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)', '2036 Annual Production (MWh)', '2036 CO2 Emissions (metric tons)']

fossil_fuel_reduction_df['2050 Annual Production (MWh)'] = 0
fossil_fuel_reduction_df['2050 CO2 Emissions (metric tons)'] = 0
```


```python
# electricity to be produced by non-fossil fuels
non_fossil_fuel_electricity_2036 = (fuel_production_by_type['Annual Production (MWh)'].sum() * 1.5) - fossil_fuel_reduction_df['2036 Annual Production (MWh)'].sum()
non_fossil_fuel_electricity_2050 = fuel_production_by_type['Annual Production (MWh)'].sum() * 2
```


```python
clean_fuels_growth_df['2050 Annual Production (MWh)'] = percent_of_clean_fuel_df['Percent of Clean Fuel'] * clean_fuel_production_2050
clean_fuels_growth_df['2050 CO2 Emissions (metric tons)'] = clean_fuels_growth_df['2050 Annual Production (MWh)'] * fuel_type_efficiency['CO2 emissions metric tons/MWh']

shift_to_non_fossil_df = grid_growth_df_no_greening.loc[['Nuclear', 'Wind', 'Hydro', 'Solar', 'Biomass', 'Geothermal']][['2021 Annual Production (MWh)', '2021 CO2 Emissions (metric tons)']]

shift_to_non_fossil_df['2036 Annual Production (MWh)'] = percent_of_clean_fuel_df['Percent of Clean Fuel'] * non_fossil_fuel_electricity_2036
shift_to_non_fossil_df['2036 CO2 Emissions (metric tons)'] = shift_to_non_fossil_df['2036 Annual Production (MWh)'] * fuel_type_efficiency['CO2 emissions metric tons/MWh']

shift_to_non_fossil_df['2050 Annual Production (MWh)'] = percent_of_clean_fuel_df['Percent of Clean Fuel'] * non_fossil_fuel_electricity_2050
shift_to_non_fossil_df['2050 CO2 Emissions (metric tons)'] = shift_to_non_fossil_df['2050 Annual Production (MWh)'] * fuel_type_efficiency['CO2 emissions metric tons/MWh']
```

#### Estimated emissions


```python
shift_to_non_fossil_df = pd.concat([fossil_fuel_reduction_df, shift_to_non_fossil_df])
shift_to_non_fossil_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>2021 Annual Production (MWh)</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 Annual Production (MWh)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 Annual Production (MWh)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
    <tr>
      <th>fuel</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Gas</th>
      <td>1.6e+09</td>
      <td>2.5e+09</td>
      <td>1.2e+09</td>
      <td>7.9e+08</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Coal</th>
      <td>9e+08</td>
      <td>6.7e+08</td>
      <td>3.4e+08</td>
      <td>4.5e+08</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Oil</th>
      <td>2.6e+07</td>
      <td>4.9e+07</td>
      <td>2.4e+07</td>
      <td>1.3e+07</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Other fossil fuels</th>
      <td>1.9e+07</td>
      <td>7.2e+05</td>
      <td>3.6e+05</td>
      <td>9.7e+06</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Unknown/purchased</th>
      <td>4.2e+06</td>
      <td>9.9e+05</td>
      <td>5e+05</td>
      <td>2.1e+06</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Nuclear</th>
      <td>7.8e+08</td>
      <td>1.1e+03</td>
      <td>2.2e+09</td>
      <td>3.3e+03</td>
      <td>4e+09</td>
      <td>5.9e+03</td>
    </tr>
    <tr>
      <th>Wind</th>
      <td>3.8e+08</td>
      <td>0</td>
      <td>1.1e+09</td>
      <td>0</td>
      <td>2e+09</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Hydro</th>
      <td>2.5e+08</td>
      <td>7.8e+02</td>
      <td>7.1e+08</td>
      <td>2.3e+03</td>
      <td>1.3e+09</td>
      <td>4.1e+03</td>
    </tr>
    <tr>
      <th>Solar</th>
      <td>1.1e+08</td>
      <td>0</td>
      <td>3.3e+08</td>
      <td>0</td>
      <td>6e+08</td>
      <td>0</td>
    </tr>
    <tr>
      <th>Biomass</th>
      <td>5.4e+07</td>
      <td>1.1e+05</td>
      <td>1.5e+08</td>
      <td>3.1e+05</td>
      <td>2.8e+08</td>
      <td>5.6e+05</td>
    </tr>
    <tr>
      <th>Geothermal</th>
      <td>1.6e+07</td>
      <td>7.6e+05</td>
      <td>4.6e+07</td>
      <td>2.2e+06</td>
      <td>8.3e+07</td>
      <td>3.9e+06</td>
    </tr>
  </tbody>
</table>
</div>




```python
phase_out_fossil_fuels_chart = make_emissions_stack_chart(shift_to_non_fossil_df, 0, 3.4 * 10**9, 'Expected CO2 Emissions When Phasing Out All Fossil Fuels by 2050', False)
phase_out_fossil_fuels_chart_detail = make_emissions_stack_chart(shift_to_non_fossil_df, 0, 5 * 10**6, 'Expected CO2 Emissions For Non-Fossil Fuels When Phasing Out All Fossil Fuels by 2050', True)

show(column(phase_out_fossil_fuels_chart, phase_out_fossil_fuels_chart_detail))
```



<div id="04000afa-4c2d-487d-9dd2-06862b8f81f2" data-root-id="p506603" style="display: contents;"></div>





### Model comparison


```python
models = [grid_growth_df_no_greening, grid_growth_df_with_greening, shift_to_non_fossil_df]
emissions_years_cols = ['2021 CO2 Emissions (metric tons)', '2036 CO2 Emissions (metric tons)', '2050 CO2 Emissions (metric tons)']
emissions_years = [2021, 2036, 2050]

pd.DataFrame({
    'Model': ['No change in fuel makeup', 'No increase in fossil fuels', 'Phase out fossil fuels by 2050'],
    '2021 CO2 Emissions (metric tons)': [model['2021 CO2 Emissions (metric tons)'].sum() for model in models],
    '2036 CO2 Emissions (metric tons)': [model['2036 CO2 Emissions (metric tons)'].sum() for model in models],
    '2050 CO2 Emissions (metric tons)': [model['2050 CO2 Emissions (metric tons)'].sum() for model in models]
})
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Model</th>
      <th>2021 CO2 Emissions (metric tons)</th>
      <th>2036 CO2 Emissions (metric tons)</th>
      <th>2050 CO2 Emissions (metric tons)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>No change in fuel makeup</td>
      <td>3.2e+09</td>
      <td>4.8e+09</td>
      <td>6.4e+09</td>
    </tr>
    <tr>
      <th>1</th>
      <td>No increase in fossil fuels</td>
      <td>3.2e+09</td>
      <td>3.2e+09</td>
      <td>3.2e+09</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Phase out fossil fuels by 2050</td>
      <td>3.2e+09</td>
      <td>1.3e+09</td>
      <td>4.5e+06</td>
    </tr>
  </tbody>
</table>
</div>



NB: the second model (no increase in fossil fuels) appears to have no change in emissions in this table due to the scientific notation. See the actual numbers below.


```python
print('Emissions numbers for second model (2021, 2036, 2050):')
grid_growth_df_with_greening['2021 CO2 Emissions (metric tons)'].sum(), grid_growth_df_with_greening['2036 CO2 Emissions (metric tons)'].sum(), grid_growth_df_with_greening['2050 CO2 Emissions (metric tons)'].sum()
```

    Emissions numbers for second model (2021, 2036, 2050):





    (3208480229.82296, 3209610119.247571, 3210740008.672182)




```python
print('Increase in emissions from 2021-2050 without additional fossil fuels:')
grid_growth_df_with_greening['2050 CO2 Emissions (metric tons)'].sum() - grid_growth_df_with_greening['2021 CO2 Emissions (metric tons)'].sum()
```

    Increase in emissions from 2021-2050 without additional fossil fuels:





    2259778.849222183




```python
emissions_growth_data_chart = figure(width=800, height=400)

emissions_growth_data_chart.line(emissions_years, [grid_growth_df_no_greening[yr].sum() for yr in emissions_years_cols], line_color=Category10[3][0], legend_label='No change in fuel makeup')
emissions_growth_data_chart.line(emissions_years, [grid_growth_df_with_greening[yr].sum() for yr in emissions_years_cols], line_color=Category10[3][1], legend_label='No increase in fossil fuels')
emissions_growth_data_chart.line(emissions_years, [shift_to_non_fossil_df[yr].sum() for yr in emissions_years_cols], line_color=Category10[3][2], legend_label='Phase out fossil fuels by 2050')

emissions_growth_data_chart.title = 'CO2 emissions predictions, 2021-2050'
emissions_growth_data_chart.legend.orientation = 'vertical'
emissions_growth_data_chart.legend.background_fill_color = '#fafafa'
emissions_growth_data_chart.add_layout(emissions_growth_data_chart.legend[0], 'right')

show(emissions_growth_data_chart)
```



<div id="d664ded8-0592-4538-ae69-61ab4185d47c" data-root-id="p733157" style="display: contents;"></div>





From these models, we can see that:
    
- If we expand our grid without changing its fuel makeup, we will double our emissions by 2050 
- Not increasing fossil fuels would greatly slow the rate of emissions growth, with an increase of 2.25 million tons annually by 2050
- The only path to carbon neutrality requires phasing out our fossil fuels
- We also need to make our non-fossil fuels more efficient and less polluting, because all non-fossil fuels (except wind and solar) come with a carbon footprint
- In addition to decreasing the carbon footprint of fuels like geothermal energy, we should also be looking at increasing the percent of electricity produced by fuels that already have a small to zero carbon footprint, such as solar, wind, and nuclear power 

## Conclusion

If the US is to reduce carbon emissions from power plants, the number one priority must be to **eliminate gas-fueled power plants**. Gas produces 94% of the carbon emissions from power production despite accounting for only 41% of power production. Every reduction in gas-produced electricity will pay off more than twice over in carbon emissions reductions. 

Because gas-fueled power plants account for a plurality of power produced in the US, eliminating gas (and other fossil fuels) from the power grid must be paired with **investing in a power grid driven by non-fossil fuels**. With electricity demand expected to double within the next 30 years, more clean(er) power production is critical to meeting demand while also reaching net zero emissions.

Finally, moving away from fossil fuels and using more non-fossil fuels is not enough to move the US to net zero emissions from electricity. Fuels such as geothermal and nuclear energy, while producing miniscule emissions levels in comparison to gas, oil, and coal, still emit CO2 into our atmosphere. Without technological improvements to make these fuel sources carbon-free, we will continue to create carbon emissions from electricity production and will be unable to reach net zero emissions by 2050.

## Ideas for future investigation:
    
- How many power plants will we need to build to move off of fossil fuels while meeting increased electricity demand?
- Can any of our fossil fuel power plants be converted into non-fossil fuel plants?
- Are there particular power companies/states producing power with a lower emissions rate that could serve as models for cleaner power production?
- How much extra electricity could we generate from our existing grid, whether through less downtime or improvements to generators? 
- How will changing climate impact what sort of fuels we can use and where power plants can be built? For example. as water becomes more scarce, how viable will hydroelectric plants be?
- How much electricity is generated by solar/wind/geothermal generators attached to individual buildings? Is it practical to use a "personal power plant" system to power buildings?
- Nuclear power is already a well-established part of our electric grid, producing nearly 1/5 of US electricity. What safety and/or political concerns exist around expanding our use of nuclear power?
