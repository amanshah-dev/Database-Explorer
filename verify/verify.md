create function verify(token text, secret text, algorithm text DEFAULT 'HS256'::text)
    returns TABLE(header json, payload json, valid boolean)
    immutable
    language sql
as
$$
  SELECT
    convert_from(url_decode(r[1]), 'utf8')::json AS header,
    convert_from(url_decode(r[2]), 'utf8')::json AS payload,
    r[3] = algorithm_sign(r[1] || '.' || r[2], secret, algorithm) AS valid
  FROM regexp_split_to_array(token, '\.') r;
$$;

alter function verify(text, text, text) owner to learner;
