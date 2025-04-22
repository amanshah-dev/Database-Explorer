create function sign(payload json, secret text, algorithm text DEFAULT 'HS256'::text) 
    returns text 
    immutable 
    language sql 
as 
$$
WITH
  header AS (
    SELECT url_encode(convert_to('{"alg":"' || algorithm || '","typ":"JWT"}', 'utf8')) AS data
  ),
  payload AS (
    SELECT url_encode(convert_to(payload::text, 'utf8')) AS data
  ),
  signables AS (
    SELECT header.data || '.' || payload.data AS data FROM header, payload
  )
SELECT
    signables.data || '.' ||
    algorithm_sign(signables.data, secret, algorithm) FROM signables;
$$;

alter function sign(json, text, text) owner to learner;
