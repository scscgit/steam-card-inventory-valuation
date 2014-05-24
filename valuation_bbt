<?php

/**********************************************************************************************
 * Steam Card Inventory Valuation v0.1 -- by Julian Fietkau (http://www.julian-fietkau.de/)
 **********************************************************************************************
 * Copyright (c) 2013, Julian Fietkau
 * 
 * Permission to use, copy, modify, and/or distribute this software for any purpose
 * with or without fee is hereby granted, provided that the above copyright notice
 * and this permission notice appear in all copies.
 * 
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH
 * REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND
 * FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT,
 * OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA
 * OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION,
 * ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 **********************************************************************************************
 * Information for server administrators:
 * - This script should be very simple to install -- just put it somewhere where there's
 *   also PHP. I've tested only on PHP 5, but I think it doesn't do anything exotic.
 * - Please create a directory named "cache" in the same directory where the script resides
 *   and make sure that PHP has read and write access. It will create many files, one for
 *   each card that is requested. Since each of those is about 100KB large, they'll quickly
 *   run into the tens (maybe hundreds) of MB. A fairly easy way to reduce the footprint would
 *   be to cache just the price instead of the full Community Market HTML, if you're inclined
 *   to optimize.
 * - For each un-cached card, the script will make an HTTP request to Valve. You can adjust
 *   the delay between requests and the cache retention time below. Please note that some
 *   use cases for this script may conflict with Valve's terms of use for the Steam website,
 *   so in your own interest, please tread carefully.
 * - Exchange rates are scraped from Bloomberg. The results are cached as well.
 **/

/* Server configuration, these things should be adjusted by the server operator. */

// Delay between HTTP requests to the Community Market, in milliseconds. Default is 100.
// Higher values will slow down requests for large card inventories, but lower values
// will increase traffic and could potentially lead to consequences if Valve notices.
define('REQUEST_DELAY', 100);

// Cache retention time in seconds, this is the maximum age of a market listing (and
// thus card valuation) before it is refreshed from Valve's servers. Default is 24h.
define('CACHE_RETENTION', 24*60*60);

/* Set user parameters: profile and currency */

// defaults
$target_currency = 'EUR';
$steam_profile = null;
$terms_accepted = false;

if(isset($_GET['cur'])) {
  $user_currency = strtoupper($_GET['cur']);
  if(in_array($user_currency, array('USD', 'EUR', 'GBP', 'AUD', 'RUB', 'BRL'))) {
    $target_currency = $user_currency;
  }
}

if(isset($_GET['terms'])) {
  $terms_accepted = true;
}

if(isset($_GET['profile'])) {
  $user_profile = $_GET['profile'];
  // How do you validate a Steam profile name? What characters are allowed?
  // Let's simply exclude slashes and quotes and hope for the best.
  if(strpos($user_profile, '/') === false && strpos($user_profile, '"') === false
     && strpos($user_profile, '\'') === false) {
    $steam_profile = $user_profile;
  }
}

?>
<html>
<head>
<title>Steam Card Inventory Valuation</title>
<style type="text/css">
body {
  margin: 1em 3em;
  background-color: #ccc;
  font-family: Arial, Helvetica, sans-serif;
}
input[type=text] {
  border: 1px solid #800;
  padding: .3em;
}
#profile {
  margin-right: 2em;
}
.message {
  margin: 1em 0;
  padding: 1em;
}
.terms {
  border: 2px solid #800;
  background-color: #eee;
  font-size: small;
}
.message label {
  font-size: normal;
}
.message p {
  margin: 0 0 .3em;
}
input[type=submit] {
  display: block;
  margin: 0 auto;
  padding: .3em;
  font-size: large;
  font-weight: bold;
}
table {
  margin: 1em auto 0;
}
th {
  background-color: #800;
  color: #fff;
  text-align: left;
  padding: .5em;
}
td {
  background-color: #ddd;
  padding: .5em;
}
td:first-child {
  padding: 0;
}
#footer {
  text-align: center;
  font-size: smaller;
  color: #888;
}
.error {
  margin: 1em 0;
  border: 2px solid #f00;
  background-color: #fff;
}
ol {
  margin: 0 0 0 2em;
  padding: 0;
}
li {
  margin-top: .5em;
}
</style>
</head>
<body>
<h1>Steam Card Inventory Valuation</h1>
<?php

/* This function is intended to scrape a value from an HTML file, though it can be
   used for any kind of string. It returns the substring between $delimiter_left
   and $delimiter_right (first occurence). */

function extract_value($data, $delimiter_left, $delimiter_right) {
  $start_pos = strpos($data, $delimiter_left) + strlen($delimiter_left);

  if($start_pos === false) {
    // not found
    return false;
  }
  $end_pos = strpos($data, $delimiter_right, $start_pos);
  if($end_pos === false) {
    // not found
    return false;
  }

  return substr($data, $start_pos, ($end_pos - $start_pos));
}

/* This function implements our local caching, which is used to reduce
   the load on external services as well as speed things up. */

function access_cached($url, $filename) {
  // We save our data in flat files for portability, but it should be trivial
  // to drop in something like Redis instead.
  $filename = 'cache/'.$filename;
  if(file_exists($filename)) {
    $file_modified = filemtime($filename);
    if(time() - $file_modified > CACHE_RETENTION) {
      unlink($filename);
    }
  }
  if(file_exists($filename)) {
    $result = file_get_contents($filename);
  } else {
    $result = file_get_contents($url);
    file_put_contents($filename, $result);
    usleep(REQUEST_DELAY * 1000);
  }

  return $result;
}

/* This function provides an estimate for the value of a given card. */

function get_card_price($card_hash, $target_currency = false) {
  /* Get the market listing. */
  $listing_url = 'http://steamcommunity.com/market/listings/238460/'
                                     .rawurlencode($card_hash);
  $market_html = access_cached($listing_url, 'market_'.rawurlencode($card_hash).'.html');

  /* Assessing the trading value of a card is difficult, we'll just go with
     the currently cheapest listing on the Community Market (fees excluded). */

  /* Find the cheapest available listing and extract the price without fees. */

  $delimiter_left = 'market_listing_price_without_fee">';
  $delimiter_right = '</span>';

  $price_string = extract_value($market_html, $delimiter_left, $delimiter_right);
  if($price_string === false) {
    return false;
  } else {
    $price_string = trim($price_string);
  }

  /* Here is where we have to parse different currencies. The Market always returns
     the seller's native currency, so we have to do the conversion, if necessary. */

  $currency = '?'; // default
  if(strpos($price_string, ' USD') !== false) {
    $currency = 'USD';
    $amount = substr($price_string, 5, strpos($price_string, ' ') - 5);
  }
  if(strpos($price_string, '&#8364;') !== false) {
    $currency = 'EUR';
    $amount = substr($price_string, 0, strpos($price_string, '&#8364;'));
  }
  if(strpos($price_string, '&#163;') !== false) {
    $currency = 'GBP';
    $amount = substr($price_string, strpos($price_string, '&#163;') + 6);
  }
  if(strpos($price_string, 'p&#1091;&#1073;.') !== false) {
    $currency = 'RUB';
    $amount = substr($price_string, 0, strpos($price_string, 'p&#1091;&#1073;.') - 1);
  }
  if(strpos($price_string, '&#82;&#36;') !== false) {
    $currency = 'BRL';
    $amount = substr($price_string, strpos($price_string, '&#82;&#36;') + 10);
  }

  /* This is where the conversion happens. */
  if($target_currency) {

    /* Unify the decimal point. This is probably not a clean solution, but works for
       my test cases. */

    $amount = str_replace(',', '.', $amount);
    $amount = floatval($amount);

    /* Get the exchange rates. */
    $factorUSDto = array();
    $factorUSDto['USD'] = 1.0;
    foreach(array('EUR', 'GBP', 'AUD', 'RUB', 'BRL') as $exchange_currency) {
      $exchange_rate_html =
        access_cached('http://www.bloomberg.com/quote/USD'.$exchange_currency.':CUR',
                      'exchange_rate_'.$exchange_currency.'.html');

      $delimiter_left = '<span class=" price">';
      $delimiter_right = '<span class="currency_factor_description">';

      $exchange_rate = extract_value($exchange_rate_html, $delimiter_left, $delimiter_right);
      if($exchange_rate === false) {
        return false;
      } else {
        $exchange_rate = trim($exchange_rate);
      }
       
      $factorUSDto[$exchange_currency] = floatval($exchange_rate);
    }

    /* Convert to target currency and round to 2 digits after the seperator. */
    $amount = $amount * $factorUSDto[$target_currency] / $factorUSDto[$currency];
    $currency = $target_currency;
    $formatted_amount = number_format($amount, 2, '.', '');
    // PHP can do fancy money formatting. TODO maybe do that here?
  }

  return array('amount' => $amount, 'formatted_amount' => $formatted_amount, 'currency' => $currency);
}

/* ---- Program flow starts here. ---- */

?>

<form method="GET" action="<?php echo basename(__FILE__); ?>">
<label for="profile">Steam profile/ID:</label>
<input type="text" name="profile" id="profile" <?php echo $steam_profile ? 'value="'.$steam_profile.'" ' : ''; ?>/>
<label for="cur">Currency:</label>
<select name="cur" size="1">
<?php
foreach(array('USD', 'EUR', 'GBP', 'AUD', 'RUB', 'BRL') as $currency) {
  echo '<option'.(($target_currency == $currency) ? ' selected' : '').'>'.$currency."</option>\n";
}
?>
</select>
<div class="message terms">
<p style="text-align:center; color:red">Forked version for Battleblock Theater</p>
<p>The inventory of the given Steam profile has to be public. If yours isn't, you can briefly change it to public, do the valuation, and then change it back.</p>
<p>The value is derived from the currently cheapest Community Market listing for any given card. If a card has no listings at all, its value is output as &quot;unknown&quot;. The value that is shown does <strong>not</strong> contain the transaction fees, i.e. it is 15% lower than what you actually see on the Community Market. It is done this way so that you see what you could potentially gain by selling the cards right now, not what the buyer would pay.</p>
<p>There may be slight inaccuracies in the currency conversions due to rounding. For that matter, pricing values may be wrong for any number of as-yet unknown reasons. If you see unusual results, please don't take them at face value and double-check before trading/selling.</p>
<p>Market listings are cached for 24 hours. Your first request may take a long time to complete, but any subsequent ones on the same day should be faster. If it seems like your browser is loading for a very long time, please be patient. If you get a timeout error, please refresh -- the program will resume from where it failed. If you have an extremely large amount of cards, you may have to do this several times.</p>
<p>ANY USE OF THIS WEBSITE OR ITS CONTENTS OCCURS STRICTLY AT YOUR OWN RISK.</p>
<p><input type="checkbox" name="terms" <?php echo $terms_accepted ? 'checked="checked" ' : ''; ?>/>
<label for="terms">I have understood those things.</label></p>
</div>
<input type="submit" value="Calculate!" />
</form>

<?php

if($steam_profile != null && $terms_accepted) {

  /* Retrieve Steam Community inventory (cards, backgrounds, emoticons...) for given user */

  $cards_json = file_get_contents('http://steamcommunity.com/profiles/'
                                .$steam_profile.'/inventory/json/238460/2');
  // If the profile can not be accessed, Steam will send an HTML error instead of JSON
  if(strpos($cards_json, '<html') !== false) {
    // try again as display name instead of profile ID
    $cards_json = file_get_contents('http://steamcommunity.com/id/'
                                .$steam_profile.'/inventory/json/238460/2');
  }
  if(strpos($cards_json, '<html') !== false) { // still no success?
    $error = true;
?>

<div class="message error">
<p>There was a problem trying to access the inventory.</p>
<ol>
<li>Have you entered your Steam profile/ID correctly? It should be the part that appears in your profile URL.</li>
<li><a href="http://issteamdown.com/">Is Steam down?</a></li>
<li>Sorry, sometimes you just have to try again later.</li>
</ol>
</div>

<?php
  }

  $cards = json_decode($cards_json, $assoc = true);
  unset($cards_json); // this could be large, so free up some memory

  /* The inventory data is split in two sections. rgInventory contains a list of all
     inventory items including doubles, rgDescriptions contains all sorts of data but only
     once per unique item. */

  /* First we extract relevant information for all cards. */

  $cards_data = $cards['rgDescriptions'];
  $cards_info = array();
echo($cards[1]);
  foreach($cards_data as $card_data) {
    // filter everything that is not a card
    $is_card = true;
    foreach($card_data['tags'] as $tag) {
      if($tag['name'] == 'Trading Card') {
        $is_card = true;
      }
    }
    if($is_card) {
      $card_info = array();
      $card_info['name'] = $card_data['name'];
      $card_info['market_hash_name'] = $card_data['market_hash_name'];
      $card_info['icon_url'] = $card_data['icon_url'];
      $card_info['game_name'] = $card_data['tags'][0]['name'];
      $card_info['price'] = get_card_price($card_data['market_hash_name'], $target_currency);
      $cards_info[$card_data['classid']] = $card_info;
    }
  }

  /* Then we cross-reference the data with the inventory to create a list of all cards. */

  $cards_inventory = $cards['rgInventory'];
  $cards_list = array();
  foreach($cards_inventory as $card_inventory_entry) {
    $card_hash = $cards_info[$card_inventory_entry['classid']]['market_hash_name'];
    // we use this to lazily filter non-card items again, their classid
    // will not be in $cards_info.
    if($card_hash) {
      $cards_list[] = $cards_info[$card_inventory_entry['classid']];
    }
  }

  /* Final list of cards is in $cards_list and will be output later. */

  if(!$error) {
?>

<hr />

<table>
<tr>
<th></th>
<th>Card</th>
<th>Game</th>
<th>Value</th>
</tr>
<?php

    $total = 0.0;

    foreach($cards_list as $card) {
      echo '<tr><td>';
      echo '<img src="http://cdn.steamcommunity.com/economy/image/'
           .$card['icon_url'].'/60x70f" />';
      echo '</td><td>';
      echo '<a href="http://steamcommunity.com/market/listings/238460/'
           .$card['market_hash_name'].'">'.$card['name'].'</a>';
      echo '</td><td>';
      echo $card['game_name'];
      echo '</td><td>';
      if($card['price'] === false) {
        echo 'unknown';
      } else {
        $total += $card['price']['amount'];
        echo $card['price']['formatted_amount'].' '.$card['price']['currency'];
      }
      echo '</td></tr>';
      echo "\n";
    }

    $formatted_total = number_format($total, 2, '.', '');

?>
<tr>
<th></th>
<th>Total</th>
<th></th>
<th><?php echo $formatted_total.' '.$target_currency; ?></th>
</tr>
</table>

<?php
  }
}

?>

<hr />

<p id="footer">Card Inventory Valuation programmed by Julian Fietkau. No affiliation (but plenty of admiration) to Valve Corporation.</p>

</body>
</html>
