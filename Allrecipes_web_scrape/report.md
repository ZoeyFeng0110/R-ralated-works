    library(rvest)
    library(dplyr)

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

    library(ggplot2)
    library(tidyr)

## Website Choose

Allrecipes is a globally renowned community-driven culinary website,
established in 1997 and originally named CookieRecipe.com. The platform
brings together home cooks from around the world to share and exchange
their recipes, cooking techniques, and experiences. I will scrape the
nutritional content data of popular beef dishes from this website.

## Scrape Process

Define a function to loop through the webpage and extract the
nutritional content of beef-related dishes that are uniformly listed and
recorded.

    scrape_nutrition_value <- function(links) {
      nutrition_values <- list()
      for (link in links) {
       page <- read_html(link)
       values <- page %>%
          html_elements("span.mm-recipes-nutrition-facts-label__nutrient-name--has-postfix") %>%  # navigate to <span>
          html_nodes(xpath = "./following-sibling::text()") %>%  # Retrieve adjacent text nodes
          html_text()
       nutrition_values <- c(nutrition_values, list(values))
      }
      return(do.call(c, nutrition_values))
    }

Define a function to loop through the webpage and extract the dish
names.

    scrape_headers <- function(links) {
      headers <- c()
      for (link in links) {
        page <- read_html(link)
        header <- page %>%
          html_elements("h1.article-heading.text-headline-400") %>%  
          html_text() 
        headers <- c(headers, header)
      }
      return(headers)
    }

Scrape the links of the selected pages (dishes with beef as the main
ingredient).

    page_one <- "https://www.allrecipes.com/recipes/200/meat-and-poultry/beef/"
    page <- read_html(page_one)
    links <- page %>%
      html_elements("#tax-sc__recirc-list_1-0 a") %>%
      html_attr("href")

Call the scraping function to perform the data extraction mentioned
above.

    headers_name <- scrape_headers(links)
    nutrition_data <- scrape_nutrition_value(links)

Scrape the types of nutrition listed for the dishes on the website to
later organize them into a table as column names for the nutritional
content of the dishes.

    page <- "https://www.allrecipes.com/beef-and-bean-tiny-tacos-recipe-8727881" %>%
      read_html(page_one)
    nutrition_name <- html_elements(page, "div.mm-recipes-nutrition-facts-label__contents table tbody tr td span") %>% 
      html_text()
    df_column <- c("Cuisine name", nutrition_name)

Organize the scraped data and process it into a table format suitable
for subsequent visualization.

    # initialize an empty dataframe
    df <- data.frame(matrix(ncol = length(df_column), nrow = ceiling(length(nutrition_data) / length(nutrition_name))))
    colnames(df) <- df_column

    # populate the first column with headers_name
    df[, 1] <- headers_name

    # fill in the remaining nutritional data
    for (i in seq_len(nrow(df))) {
      if (length(nutrition_data) >= (i - 1) * length(nutrition_name) + length(nutrition_name)) {
        # dynamically insert values (each group of values should match the length of nutrition_name)
        df[i, 2:ncol(df)] <- nutrition_data[((i - 1) * length(nutrition_name) + 1):(i * length(nutrition_name))]
      } else {
        df[i, 2:ncol(df)] <- NA  #  if there are not enough values, fill with NA
      }
    }

Process the data format of certain parts of the dataframe to prepare it
for visualization.

    for (col in 2:ncol(df)) {  # skip the first column since it represents the dish name.
      df[[col]] <- as.integer(gsub("[^0-9]", "", df[[col]]))
    }
    for (i in 2:ncol(df)) {
      first_value <- df[1, i]
      if (first_value >= 100 | first_value == 4 ) {
        colnames(df)[i] <- paste0(colnames(df)[i], "(mg)")
      } else {
        colnames(df)[i] <- paste0(colnames(df)[i], "(g)")
      }
    }
    cleaned_nutrition_df <- df
    cleaned_nutrition_df

    ##                             Cuisine name Total Fat(g) Saturated Fat(g)
    ## 1          French Onion Beef and Noodles           23                8
    ## 2 Tex-Mex Ground Beef and Potato Skillet           59               17
    ## 3               Beef and Bean Tiny Tacos           51               10
    ## 4 Bobotie (South African Beef Casserole)           27               10
    ## 5                        The Crustburger           56               24
    ## 6                Italian Steak Pizzaiola           33               11
    ## 7                    Classic Swiss Steak           22                7
    ## 8        Slow Cooker Stuffed Pepper Soup           14                5
    ##   Cholesterol(mg) Sodium(mg) Total Carbohydrate(g) Dietary Fiber(g)
    ## 1             117        536                    31                2
    ## 2             114       1698                    65                7
    ## 3              52       1024                    67                5
    ## 4             155        580                    13                2
    ## 5             167       2248                    58                4
    ## 6              88        615                    12                3
    ## 7             129       1002                    33                4
    ## 8              58        966                    18                3
    ##   Total Sugars(g) Protein(g) Vitamin C(mg) Calcium(mg) Iron(mg) Potassium(mg)
    ## 1               5         34             4         168        4           465
    ## 2               5         40            46         327        5          1409
    ## 3               1         24            26         169        3          1630
    ## 4               4         36             7         137        5           654
    ## 5              16         47             3        1358        6          1018
    ## 6               6         30            23          56        4           789
    ## 7               8         45            29          81        7           915
    ## 8               6         23            47         141        3           732

## Data Visualization & Data Analysis

-   **Comparison of the main nutritional content of each dish**

<!-- -->

    long_data <- cleaned_nutrition_df %>%
      pivot_longer(
        cols = c(`Total Fat(g)`, `Saturated Fat(g)`, `Total Carbohydrate(g)`, `Dietary Fiber(g)`, `Total Sugars(g)`, `Protein(g)`),
        names_to = "Main_Nutrient",
        values_to = "Value"
      )

    comparision <- long_data %>%
      ggplot(aes(x = Main_Nutrient, y = Value, fill = Main_Nutrient)) +
      geom_bar(stat = "identity") +
      facet_wrap(~ `Cuisine name`, scales = "free_y") +
      theme_minimal() +
      labs(
        title = "Main Nutritional Content of Dishes by Cuisine",
        x = "Main Nutrient",
        y = "Value"
      ) +
      theme(axis.text.x = element_text(angle = 45, hjust = 1, size = 6),
            strip.text = element_text(size = 7))

    print(comparision)

![](report_files/figure-markdown_strict/unnamed-chunk-8-1.png) -
**Calorie comparison between different dishes (based on fat data)**

    fat_data <- cleaned_nutrition_df %>% 
      ggplot(aes(x = `Cuisine name`, y = `Total Fat(g)`)) +
      geom_bar(stat = "identity") +
      theme_minimal() +
      labs(title = "Total Fat by Cuisine", x = "Cuisine", y = "Total Fat (g)") +
      theme(axis.text.x = element_text(angle = 45, hjust = 1))

    print(fat_data)

![](report_files/figure-markdown_strict/unnamed-chunk-9-1.png) This
chart clearly shows significant differences in fat content across the
dishes, which might have varying health implications. Dishes like
“Tex-Mex Ground Beef and Potato Skillet” and “The Crustburger” have over
50 grams of fat, likely due to the use of high-fat ingredients or
cooking methods like frying. On the other hand, “Slow Cooker Stuffed
Pepper Soup” has only 14 grams of fat, which is much healthier, possibly
because it uses low-fat ingredients or simpler cooking methods. The
dishes in the middle range, such as “Italian Steak Pizzaiola” and “Beef
and Bean Tiny Tacos,” have fat content between 20 and 40 grams, making
them relatively more balanced. These differences remind us to pay
attention to nutritional information when choosing dishes, especially
for cuisines that tend to use calorie-dense ingredients.

-   **The Healthiness Of Dishes (based on their sodium and fat
    content)**

<!-- -->

    health <- cleaned_nutrition_df %>%
      ggplot(aes(x = `Sodium(mg)`, y = `Total Fat(g)`, color = `Cuisine name`)) +
      geom_point() +
      theme_minimal() +
      labs(title = "Dish Health Assessment (Based on Fat and Sodium Content Data)", x = "The sodium content", y = "The fat content") 

    print(health)

![](report_files/figure-markdown_strict/unnamed-chunk-10-1.png)

High fat and sodium content in food can potentially pose health risks
when consumed in excessive amounts. High fat intake, particularly
saturated and trans fats, is associated with increased risks of obesity,
cardiovascular diseases, and high cholesterol levels. Similarly,
excessive sodium consumption is linked to high blood pressure, heart
disease, and stroke. Based on the scatter plot, some dishes, such as
“The Crustburger,” stand out with significantly high fat and sodium
levels, indicating they may be less healthy options. In contrast, dishes
like “Slow Cooker Stuffed Pepper Soup,” with lower fat and sodium
content, appear to be healthier choices. The chart highlights the
variation in nutritional profiles across dishes and cuisines,
emphasizing the need for mindful selection of meals to promote better
health outcomes.

## Reflection

While completing this assignment, I found the most challenging part was
organizing the scraped data into a well-structured format for storage.
Although the data volume was not large, I often found myself getting
confused when using for loops to repeatedly add data to the dataframe. I
had to rely on AI to help me plan the workflow. Additionally, during the
visualization phase, I realized I needed to go back and add more
data-cleaning steps, such as transforming data formats, removing units,
and appending them to column names. This assignment has been a great
practice for my final project, allowing me to develop clearer coding
logic and reduce redundant operations in future tasks.
