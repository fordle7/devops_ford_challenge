# Problem 1: Too Many Things To Do

## 1. Install Dependencies

```sh
sudo apt update
sudo apt install -y curl jq
```

## 2. Execute Command

```sh
jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' ./transaction-log.txt | xargs -I {} curl -s "https://example.com/api/{}" >> ./output.txt
```

### Explanation

- **Extract Order IDs**  
  ```sh
  jq -r 'select(.symbol == "TSLA" and .side == "sell") | .order_id' ./transaction-log.txt
  ```
  - Filters transactions where `symbol` is `"TSLA"` and `side` is `"sell"`, then extracts the `order_id`.

- **Submit Requests**  
  ```sh
  xargs -I {} curl -s "https://example.com/api/{}" >> ./output.txt
  ```
  - Sends an HTTP GET request to `https://example.com/api/:order_id` for each extracted order ID.
  - Appends the response to `output.txt`.
  - `-s` suppresses curl's progress output.

