# Licers for C and C++

License-key verification for **C** and **C++** apps, against [Licers](https://licers.com): games, plugins (VST/AU, Unreal, Qt), CAD tools, embedded Linux, anything that can link a C library.

Every answer from the license server is **Ed25519-signed** with your account key and checked inside the library, with a fresh random nonce per call, so a cracked "fake license server", an edited response or a replayed one is refused.

- `licers.h`: a plain C API (C99, opaque handles, status codes)
- `licers.hpp`: a header-only C++17 wrapper with RAII and exceptions
- One library, `licers.dll` / `liblicers.so` / `liblicers.dylib` (or the static `licers.lib` / `liblicers.a`), with its own TLS: no OpenSSL or other runtime dependency

It is built on the [`licers` Rust crate](../rust), so it behaves exactly like the other Licers SDKs and computes the same device id (the Python SDK has its own).

## Install

**Prebuilt package** (no Rust needed): download `licers-c-1.0.1-<platform>` from [GitHub Releases](https://github.com/mightycoders00/licers-c/releases/latest), unzip it and point CMake at it.

```cmake
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_SOURCE_DIR}/third_party/licers-c-1.0.1-windows-x64")
find_package(licers 1.0 REQUIRED)
target_link_libraries(myapp PRIVATE licers::licers)   # or licers::static
licers_copy_runtime(myapp)                            # Windows: copies licers.dll next to myapp.exe
```

Without CMake, add `include/` to your include path and link `lib/licers.dll.lib` (Windows, ship `bin/licers.dll` with your app) or `-llicers` (Linux/macOS).

Linking the static library (`licers::static` does all of this for you): define `LICERS_STATIC` before including the header, and also link
- **Windows:** `bcrypt advapi32 kernel32 ntdll userenv ws2_32 dbghelp`
- **Linux:** `pthread dl m rt`

**From source** (needs [Rust](https://rustup.rs) 1.71+):

```cmake
add_subdirectory(path/to/licers/sdks/c)
target_link_libraries(myapp PRIVATE licers::licers)
```

Your public key and product id are on the dashboard’s [Integration](https://app.licers.com/developers/integration) page. The public key is safe to ship inside your app. Each example below is a complete file.

## C++

### Check a key

Run this when your app starts. The first check binds the device to the key; past the key’s device limit a new device is refused.

In the dashboard: create a product, then a license key for it.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    auto result = client.validate(key);
    if (!result.valid()) {
        // Expired, revoked, device limit reached... the server says which.
        std::cerr << "Not licensed: " << result.error() << "\n";
        return 1;
    }

    std::cout << "Licensed. Expires: " << result.expires_at().value_or("never") << "\n";
    return 0;
} catch (const licers::Error& e) {
    if (e.is_network()) {
        // Offline, or the server is down. That is not "unlicensed": try again,
        // or allow a few days since the last good check.
        std::cerr << "Could not reach the license server. Please try again.\n";
    } else {
        // A forged or replayed answer, or a key for another of your products.
        std::cerr << "License check failed: " << e.what() << "\n";
    }
    return 1;
}
```

- A refused key is a normal answer with a message. An unreachable server is an error: never treat it as “unlicensed”.

A refused key is `valid() == false`. A forged answer or an unreachable server throws, and `e.is_network()` tells them apart.

### Entitlements

Switch features on per key: true/false, numbers or text. They come from the signed answer.

In the dashboard: set Entitlements (JSON) on the key or on its plan, for example {"pro": true, "seats": 5}.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    auto result = client.validate(key);
    if (!result.valid()) {
        std::cerr << "Not licensed: " << result.error() << "\n";
        return 1;
    }

    // Entitlements of this key, for example {"pro": true, "seats": 5, "tier": "gold"}.
    // They are read from the signed copy, so nobody can edit them on the way.
    if (result.feature_bool("pro")) std::cout << "Pro features unlocked\n";

    auto seats = result.feature_int("seats").value_or(1);
    auto tier = result.feature_string("tier").value_or("basic");
    std::cout << "Seats: " << seats << " Tier: " << tier << " All: " << result.features_json() << "\n";
    return 0;
} catch (const licers::Error& e) {
    std::cerr << "License error: " << e.what() << "\n";
    return 1;
}
```

- Compare with true exactly. The text "false" is not the same as false.

### Offline file

Check a signed .lic file with no internet at all, for air-gapped machines. Signature, product, expiry, device and operating system are all checked on the computer.

1. Your app shows its device id, and the customer sends it to you.
2. License keys → ⋯ → Offline license file: paste the device id, set how many days it lasts, download the .lic file.
3. Send it to the customer. Your app reads it from disk.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    // 1. Show this to the customer. They send it to you, and you create the file
    //    for that device: License keys -> ... -> Offline license file.
    std::cout << "Device id: " << licers::hwid() << "\n";

    // 2. The customer saves the file next to your app. No network is used here.
    auto lic = licers::verify_offline_license("YOUR_PUBLIC_KEY", "license.lic",
        "",   // this device
        "YOUR_PRODUCT_ID");   // your product

    // 3. Use it. It never returns an invalid license: it throws instead.
    std::cout << "Licensed to: " << lic.customer().value_or("-")
              << " Expires (unix time, 0 = never): " << lic.expires_at() << "\n";
    if (lic.feature_bool("pro")) std::cout << "Pro features unlocked\n";
    return 0;
} catch (const licers::SignatureError&) {
    std::cerr << "The license file is missing, damaged or was edited.\n";
    return 1;
} catch (const licers::Error& e) {
    std::cerr << "License file refused: " << e.what() << "\n";
    return 1;
}
```

- Revoking a key can’t reach a file you already sent, so give files a short life.
- The expiry uses the computer’s own clock.
- A blank device id makes a file that works on any computer.

### Usage credits

Meter what the customer does: “100 exports”, “5,000 AI calls a month”. Spending is all or nothing.

In the dashboard: give the key a credit limit, and a daily or monthly refill if you want one.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    auto result = client.validate(key);   // first: this activates the device on the key
    if (!result.valid()) {
        std::cerr << "Not licensed: " << result.error() << "\n";
        return 1;
    }

    auto balance = client.get_usage(key);   // reads only
    if (balance.remaining) std::cout << "Left: " << *balance.remaining << "\n";   // empty = no limit

    auto usage = client.record_usage(key, 1);   // spend 1 credit, all or nothing
    std::cout << "Done. Used so far: " << usage.used << "\n";
    return 0;
} catch (const licers::UsageExceeded& e) {
    std::cout << "Out of credits. Refill: " << e.reported().resets_at << "\n";
    return 1;
} catch (const licers::Error& e) {
    std::cerr << "License error: " << e.what() << "\n";
    return 1;
}
```

- Check the key on this device first.
- After a network error the credits may already be spent: read the balance before trying again.

### Remote variables

Values you change in the dashboard without shipping an update: a link, a message, a switch.

In the dashboard: add them on the Remote Variables page.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    auto result = client.validate(key);   // first: this activates the device on the key
    if (!result.valid()) {
        std::cerr << "Not licensed: " << result.error() << "\n";
        return 1;
    }

    // Values you set on the Remote Variables page. Signed, and only for valid keys.
    auto url = client.get_variable(key, "download_url");                   // empty when it isn't set
    auto theme = client.get_variable(key, "theme").value_or("dark");       // or your own default
    std::cout << url.value_or("-") << " " << theme << "\n";

    for (const auto& [name, value] : client.get_variables(key))            // all this key may see
        std::cout << name << " = " << value << "\n";
    return 0;
} catch (const licers::Error& e) {
    std::cerr << "License error: " << e.what() << "\n";
    return 1;
}
```

- Check the key on this device first.
- Values are text. A product’s value wins over one for all products.

### Floating seats

Let only N people run the app at the same time, on any computers.

In the dashboard: set Max concurrent sessions on the key.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    // Takes a seat and keeps it with heartbeats on a background thread. It is freed
    // when the variable goes out of scope.
    auto seat = client.start_session(key, [](const std::string& reason) {
        std::cerr << "Seat lost: " << reason << "\n";   // save the user's work
    });
    std::cout << "Seat taken: " << seat.seats_used() << " of " << seat.seats_max() << "\n";

    // ... your app runs here. Call this at safe points (before a save, between
    // jobs). It throws once the seat is gone.
    seat.ensure_active();
    return 0;
} catch (const licers::SeatUnavailable& e) {
    std::cerr << "No seat: " << e.what() << "\n";
    return 1;
} catch (const licers::Error& e) {
    std::cerr << "License error: " << e.what() << "\n";
    return 1;
}
```

- If the server can’t be reached, the seat is kept until its last signed expiry and then given up.
- A crashed app frees its seat by itself after the session timeout.

### Updates

Tell the customer about a new version. Only a valid, active key is given the download link.

In the dashboard: add the version and its download link on the Releases page.

```cpp
#include "licers.hpp"
#include <iostream>

int main() try {
    licers::Options opts;
    opts.product_id = "YOUR_PRODUCT_ID";   // your product: refuses keys for your other products
    opts.app_version = "2.4.1";   // shows in Analytics -> App versions
    licers::Client client("YOUR_PUBLIC_KEY", opts);   // safe to ship inside your app
    const std::string key = "XXXX-XXXX-XXXX-XXXX";   // the customer's license key

    auto update = client.check_update(key, "2.4.1", "stable");   // or "beta"

    if (update.available) {
        std::cout << "New version: " << update.version.value_or("") << "\n";
        std::cout << "What's new: " << update.notes.value_or("") << "\n";
        // The answer is signed, but download_url and notes are not: check the URL is
        // on your own host, and verify the file against a hash you publish.
        std::cout << "Download: " << update.download_url.value_or("") << "\n";
    } else {
        std::cout << "You have the latest version.\n";
    }
    return 0;
} catch (const licers::Error& e) {
    std::cerr << "License error: " << e.what() << "\n";
    return 1;
}
```

- The answer is signed, including “no update”. The download link is not: check its host and the file’s hash.

## C

Functions return a `licers_status`. `licers_validate` returns `LICERS_OK` with `licers_validation_valid() == 0` when the server refuses the key; any other status means there is no answer you can trust, and `licers_last_error()` has the message (per thread, until your next call). `const char *` strings belong to the handle they came from; `char *` results are yours to free with `licers_string_free()`.

### Check a key

Run this when your app starts. The first check binds the device to the key; past the key’s device limit a new device is refused.

In the dashboard: create a product, then a license key for it.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    int licensed = 0;
    licers_validation *result = NULL;
    licers_status rc = licers_validate(client, key, &result);

    if (rc == LICERS_ERR_NETWORK || rc == LICERS_ERR_SERVER) {
        /* Offline, or the server is down. That is not "unlicensed": try again,
           or allow a few days since the last good check. */
        fprintf(stderr, "Could not reach the license server. Please try again.\n");
    } else if (rc != LICERS_OK) {
        /* A forged or replayed answer, or a key for another of your products. */
        fprintf(stderr, "License check failed: %s\n", licers_last_error());
    } else if (!licers_validation_valid(result)) {
        /* Expired, revoked, device limit reached... the server says which. */
        fprintf(stderr, "Not licensed: %s\n", licers_validation_error(result));
    } else {
        const char *expires = licers_validation_expires_at(result);
        printf("Licensed. Expires: %s\n", expires ? expires : "never");
        licensed = 1;
    }

    licers_validation_free(result);
    licers_client_free(client);
    return licensed ? 0 : 1;
}
```

- A refused key is a normal answer with a message. An unreachable server is an error: never treat it as “unlicensed”.

### Entitlements

Switch features on per key: true/false, numbers or text. They come from the signed answer.

In the dashboard: set Entitlements (JSON) on the key or on its plan, for example {"pro": true, "seats": 5}.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    licers_validation *result = NULL;
    if (licers_validate(client, key, &result) != LICERS_OK || !licers_validation_valid(result)) {
        const char *why = result ? licers_validation_error(result) : licers_last_error();
        fprintf(stderr, "Not licensed: %s\n", why ? why : "unknown");
        licers_validation_free(result);
        licers_client_free(client);
        return 1;
    }

    /* Entitlements of this key, for example {"pro": true, "seats": 5, "tier": "gold"}.
       They are read from the signed copy, so nobody can edit them on the way. */
    if (licers_validation_feature_bool(result, "pro")) printf("Pro features unlocked\n");

    int64_t seats = 1;   /* stays 1 when the key has no "seats" */
    licers_validation_feature_int(result, "seats", &seats);
    const char *tier = licers_validation_feature_string(result, "tier");
    printf("Seats: %lld Tier: %s All: %s\n", (long long)seats, tier ? tier : "basic", licers_validation_features_json(result));

    licers_validation_free(result);
    licers_client_free(client);
    return 0;
}
```

- Compare with true exactly. The text "false" is not the same as false.

### Offline file

Check a signed .lic file with no internet at all, for air-gapped machines. Signature, product, expiry, device and operating system are all checked on the computer.

1. Your app shows its device id, and the customer sends it to you.
2. License keys → ⋯ → Offline license file: paste the device id, set how many days it lasts, download the .lic file.
3. Send it to the customer. Your app reads it from disk.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    /* 1. Show this to the customer. They send it to you, and you create the file
          for that device: License keys -> ... -> Offline license file. */
    printf("Device id: %s\n", licers_hwid());

    /* 2. The customer saves the file next to your app. No network is used here. */
    licers_offline *lic = NULL;
    licers_status rc = licers_verify_offline("YOUR_PUBLIC_KEY", "license.lic",
        NULL,   /* this device */
        "YOUR_PRODUCT_ID",   /* your product */
        &lic);
    if (rc == LICERS_ERR_SIGNATURE) {
        fprintf(stderr, "The license file is missing, damaged or was edited.\n");
        return 1;
    }
    if (rc != LICERS_OK) {
        fprintf(stderr, "License file refused: %s\n", licers_last_error());   /* expired, other device, OS or product */
        return 1;
    }

    /* 3. Use it. An invalid license is never returned. */
    const char *customer = licers_offline_string(lic, "customer");
    printf("Licensed to: %s Expires (unix time, 0 = never): %lld\n", customer ? customer : "-", (long long)licers_offline_expires_at(lic));
    if (licers_offline_feature_bool(lic, "pro")) printf("Pro features unlocked\n");

    licers_offline_free(lic);
    return 0;
}
```

- Revoking a key can’t reach a file you already sent, so give files a short life.
- The expiry uses the computer’s own clock.
- A blank device id makes a file that works on any computer.

### Usage credits

Meter what the customer does: “100 exports”, “5,000 AI calls a month”. Spending is all or nothing.

In the dashboard: give the key a credit limit, and a daily or monthly refill if you want one.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    /* First: this activates the device on the key. */
    licers_validation *result = NULL;
    if (licers_validate(client, key, &result) != LICERS_OK || !licers_validation_valid(result)) {
        const char *why = result ? licers_validation_error(result) : licers_last_error();
        fprintf(stderr, "Not licensed: %s\n", why ? why : "unknown");
        licers_validation_free(result);
        licers_client_free(client);
        return 1;
    }
    licers_validation_free(result);

    licers_usage usage;
    if (licers_get_usage(client, key, &usage) == LICERS_OK)   /* reads only; -1 = no limit */
        printf("Left: %lld\n", (long long)usage.remaining);

    licers_status rc = licers_record_usage(client, key, 1, &usage);   /* spend 1 credit, all or nothing */
    if (rc == LICERS_OK) printf("Done. %lld credits left\n", (long long)usage.remaining);
    else if (rc == LICERS_ERR_USAGE_EXCEEDED) printf("Out of credits. Refill: %s\n", usage.resets_at);
    else fprintf(stderr, "%s\n", licers_last_error());

    licers_client_free(client);
    return rc == LICERS_OK ? 0 : 1;
}
```

- Check the key on this device first.
- After a network error the credits may already be spent: read the balance before trying again.

### Remote variables

Values you change in the dashboard without shipping an update: a link, a message, a switch.

In the dashboard: add them on the Remote Variables page.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    /* First: this activates the device on the key. */
    licers_validation *result = NULL;
    if (licers_validate(client, key, &result) != LICERS_OK || !licers_validation_valid(result)) {
        const char *why = result ? licers_validation_error(result) : licers_last_error();
        fprintf(stderr, "Not licensed: %s\n", why ? why : "unknown");
        licers_validation_free(result);
        licers_client_free(client);
        return 1;
    }
    licers_validation_free(result);

    /* Values you set on the Remote Variables page. Signed, and only for valid keys. */
    char *url = NULL;   /* stays NULL when it isn't set */
    if (licers_get_variable(client, key, "download_url", &url) == LICERS_OK)
        printf("download_url = %s\n", url ? url : "(not set)");
    licers_string_free(url);

    licers_variables *vars = NULL;   /* all this key may see */
    if (licers_get_variables(client, key, &vars) == LICERS_OK) {
        for (size_t i = 0; i < licers_variables_count(vars); i++)
            printf("%s = %s\n", licers_variables_name(vars, i), licers_variables_value(vars, i));
        licers_variables_free(vars);
    }

    licers_client_free(client);
    return 0;
}
```

- Check the key on this device first.
- Values are text. A product’s value wins over one for all products.

### Floating seats

Let only N people run the app at the same time, on any computers.

In the dashboard: set Max concurrent sessions on the key.

```c
#include "licers.h"
#include <stdio.h>

/* Runs on the heartbeat thread: save the user's work. */
static void seat_lost(const char *reason, void *user_data) {
    (void)user_data;
    fprintf(stderr, "Seat lost: %s\n", reason);
}

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    /* Takes a seat and keeps it with heartbeats on a background thread. */
    licers_session *seat = NULL;
    licers_status rc = licers_session_start(client, key, seat_lost, NULL, &seat);
    if (rc != LICERS_OK) {
        fprintf(stderr, "No seat: %s\n", licers_last_error());
        licers_client_free(client);
        return 1;
    }
    printf("Seat taken: %lld of %lld\n", (long long)licers_session_seats_used(seat), (long long)licers_session_seats_max(seat));

    /* ... your app runs here. Call this at safe points (before a save, between
       jobs). It is not LICERS_OK once the seat is gone. */
    if (licers_session_ensure_active(seat) != LICERS_OK) fprintf(stderr, "Seat lost.\n");

    licers_session_free(seat);   /* frees the seat for someone else */
    licers_client_free(client);
    return 0;
}
```

- If the server can’t be reached, the seat is kept until its last signed expiry and then given up.
- A crashed app frees its seat by itself after the session timeout.

### Updates

Tell the customer about a new version. Only a valid, active key is given the download link.

In the dashboard: add the version and its download link on the Releases page.

```c
#include "licers.h"
#include <stdio.h>

int main(void) {
    licers_options opts = {0};
    opts.product_id = "YOUR_PRODUCT_ID";   /* your product: refuses keys for your other products */
    opts.app_version = "2.4.1";   /* shows in Analytics -> App versions */

    licers_client *client = NULL;   /* the public key is safe to ship inside your app */
    if (licers_client_new("YOUR_PUBLIC_KEY", &opts, &client) != LICERS_OK) {
        fprintf(stderr, "%s\n", licers_last_error());
        return 1;
    }
    const char *key = "XXXX-XXXX-XXXX-XXXX";   /* the customer's license key */

    licers_update update;
    if (licers_check_update(client, key, "2.4.1", "stable", &update) != LICERS_OK) {   /* or "beta" */
        fprintf(stderr, "%s\n", licers_last_error());
        licers_client_free(client);
        return 1;
    }

    if (update.available) {
        printf("New version: %s\n", update.version ? update.version : "");
        printf("What's new: %s\n", update.notes ? update.notes : "");
        /* The answer is signed, but download_url and notes are not: check the URL is
           on your own host, and verify the file against a hash you publish. */
        printf("Download: %s\n", update.download_url ? update.download_url : "");
    } else {
        printf("You have the latest version.\n");
    }

    licers_update_clear(&update);
    licers_client_free(client);
    return 0;
}
```

- The answer is signed, including “no update”. The download link is not: check its host and the file’s hash.

## Errors

| Error | Meaning |
|---|---|
| `LICERS_ERR_NETWORK` | Server unreachable: retry, or apply a grace period |
| `LICERS_ERR_SERVER` | The server answered 5xx: an outage, not a verdict on the key |
| `LICERS_ERR_SIGNATURE` | Signature or nonce check failed: forged, edited or replayed (or a damaged license file) |
| `LICERS_ERR_PRODUCT_MISMATCH` | Correctly signed, but for another of your products |
| `LICERS_ERR_INVALID_LICENSE` | The server (or an offline file) refused the key |
| `LICERS_ERR_USAGE_EXCEEDED` | Not enough credits; nothing was spent |
| `LICERS_ERR_SEAT_UNAVAILABLE` | No free floating seat, or a held seat was lost |
| `LICERS_ERR_PROTOCOL` | Not a usable reply (a proxy or captive portal; redirects are never followed) |

In C++ the same errors are exceptions: `NetworkError`, `ServerError`, `SignatureError` and so on, all `licers::Error`. The update check’s `download_url` and `notes` are not signed.

## Threads

Every call is a blocking network request (5 s timeout by default, set with `timeout_ms`), so keep them off your UI thread. A client may be shared by any number of threads. `on_notice` runs on the thread that called `licers_validate`. `on_lost` runs on the session's heartbeat thread, and never after `licers_session_free` returns.

## Device id

`licers_hwid()` is SHA-256 of `"licers:"` + the OS machine id (Windows `MachineGuid`, Linux `/etc/machine-id`, macOS `IOPlatformUUID`). Every Licers SDK except Python computes the same value on the same machine. Where none can be read, a random id is stored once in the user's data folder. Set `opts.hwid` to use your own.

## Tests

```sh
cmake -S . -B build -DLICERS_BUILD_TESTS=ON
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

The tests start `../testkit/fake-server.mjs` (needs Node). It signs with a real Ed25519 key and misbehaves on purpose: forged, unsigned and replayed answers, the wrong product, 5xx, redirects and non-JSON bodies. The C test runs against both the shared and the static library.

## Release packages

```sh
node scripts/package.mjs     # -> dist/licers-c-1.0.1-<os>-<arch>.zip / .tar.gz
```

Run it once on each platform you ship.

MIT licensed.
