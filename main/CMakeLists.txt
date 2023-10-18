idf_component_register(SRCS "app_main.c"
                    INCLUDE_DIRS "." REQUIRES esp_netif esp_event esp-tls esp_http_client nvs_flash protocol_examples_common)
