WS - Websocket (Server, client coming soon) Implementation for OCaml
====================================================================

The following is an example for getting set websockets set up with cohttp + Lwt. Note that this example requires cohttp >= v2.0 which is unreleased (as of writing), but you can pull and pin the master branch with:
```
$ git clone git@github.com:mirage/ocaml-cohttp.git
$ opam pin ocaml-cohttp
```

The example is also contained in `example/example_server.ml`.

**Example:**
```ocaml
open Lwt
open Cohttp
open Cohttp_lwt_unix

module Websocket = Ws.Make(Interface'_lwt.Io)

let ws_handler send =
  Some "Welcome to my websocket!" |> send
  >>= fun _ ->
  return (function
    | Some m -> Lwt_io.printf "Received message: %s\n" m
      >>= fun _ -> Some (Printf.sprintf "Thanks, I got [%s]" m) |> send
    | None -> Lwt_io.printf "Connection closed\n")

let server =
  let callback _conn req _body =
    let meth = req |> Request.meth in
    let headers = req |> Request.headers |> Header.to_list in
    match meth with
      | `GET ->
          (if Ws.is_websocket_upgrade headers then
            match Websocket.upgrade headers with
              | Error e_headers ->
                let res = Response.make ~status:`Bad_request ~headers:(e_headers |> Header.of_list) () in
                (res, fun _ oc -> Lwt_io.close oc) |> return
              | Ok ok_headers ->
                let res = Response.make ~status:`Switching_protocols ~headers:(ok_headers |> Header.of_list) () in
                (res, fun ic oc -> Websocket.handle_server ws_handler ic oc) |> return
          else
            let res = Response.make ~status:`Bad_request () in
            (res, fun _ oc -> Lwt_io.close oc) |> return)
      | _ ->
        let res = Response.make ~status:`Method_not_allowed () in
          (res, fun _ oc -> Lwt_io.close oc) |> return
  in
    Server.create ~mode:(`TCP (`Port 8000)) (Server.make_expert ~callback ())

let () = ignore (Lwt_main.run server)
```
